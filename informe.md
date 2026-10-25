# Informe Exercicio 2: Copias de seguridade e replicación en PostgreSQL

## Parte A: Copias de seguridade lóxicas

### 1. Comandos executados

**Arrancar o primario:**

```bash
docker compose up -d primario

```

**Crear a copia de seguridade:**

```bash
docker compose exec -t primario pg_dump -U bda -F c -f /var/lib/postgresql/data/albergues.dump albergues

```

**Comprobar que se fixo vendo se ocupa espazo:**

```bash
docker compose exec primario ls -lh /var/lib/postgresql/data/albergues.dump

```

**Copiar o ficheiro fóra do contedor:**

```bash
docker compose cp primario:/var/lib/postgresql/data/albergues.dump ./albergues.dump

```

* **`primario:...`**: Indica o nome do contedor/servizo e a ruta interna onde está o arquivo.
* **`./albergues.dump`**: Indica que se quere gardar na carpeta actual de WSL co mesmo nome.

**Simular o desastre (eliminar a táboa `albergues`):**

```bash
docker compose exec -t primario psql -U bda -d albergues -c "DROP TABLE IF EXISTS albergues CASCADE;"

```

**Restaurar a copia:**

```bash
docker compose exec -t primario pg_restore -U bda -d albergues -F c /var/lib/postgresql/data/albergues.dump

```

**Comprobacións finais:**

```bash
# Listar as táboas para comprobar que seguen ahí
docker compose exec -t primario psql -U bda -d albergues -c "\dt"

# Listar os albergues para comprobar que están todos
docker compose exec -t primario psql -U bda -d albergues -c "SELECT * FROM albergues;"

```

### 2. Pregunta de razoamento

**Se fixeras a copia ás 9:00 e o desastre ocorrese ás 13:00, que datos se perderían?**

> Perderíanse todos os datos que se engadiran á táboa despois de facer a copia, é dicir, neste caso todos os datos posteriores ás 9:00.

---

## Parte B: Primario e réplica

### Paso 2: Estado da replicación

No primario:

```bash
docker compose exec -t primario psql -U bda -d albergues -c "SELECT client_addr, state, sync_state, replay_lag FROM pg_stat_replication;"

```

```text
 client_addr |   state   | sync_state | replay_lag  Watch   d Detach
-------------+-----------+------------+------------
 172.18.0.3  | streaming | async      | 
(1 row)

```

Na réplica:

```bash
docker compose exec -t replica psql -U bda -d albergues -c "SELECT pg_is_in_recovery();"

```

```text
pg_is_in_recovery 
-------------------
 t
(1 row)

```

### Paso 3: Propagación de cambios

```bash
docker compose exec -t primario psql -U bda -d albergues -c "INSERT INTO albergues (nome, concello, camino, prazas, prezo_noite) VALUES ('Albergue Proba Replicacion', 'Arzúa', 'Francés', 35, 15.00);"

```

```text
INSERT 0 1

```

### Paso 4: A réplica é só de lectura

```bash
docker compose exec -t replica psql -U bda -d albergues -c "INSERT INTO albergues (nome, concello, camino, prazas, prezo_noite) VALUES ('Albergue Erroneo', 'Melide', 'Francés', 10, 10.00);"

```

```text
ERROR:  cannot execute INSERT in a read-only transaction

```

### Paso 5: Unha réplica non é unha copia

```bash
docker compose exec -t primario psql -U bda -d albergues -c "DELETE FROM albergues WHERE camino = 'Portugués';"

```

```text
DELETE 2

```

**Que pasou?**
Para comprobalo fago unha consulta á réplica:

```bash
docker compose exec -t replica psql -U bda -d albergues -c "SELECT * FROM albergues WHERE camino = 'Portugués';"

```

```text
 id | nome | concello | camino | prazas | prezo_noite 
----+------+----------+--------+--------+-------------
(0 rows)

```

> Isto significa que o borrado propagouse inmediatamente á réplica.

**Como recuperarías os datos?**
Pois dado que a réplica xa non ten os datos, a única forma de recuperalos é utilizando o arquivo de copia de seguridade (`albergues.dump`) inxectando dende fóra:

```bash
docker compose cp ./albergues.dump primario:/tmp/albergues.dump

docker compose exec -t primario pg_restore -U bda -d albergues -F c --clean --if-exists /tmp/albergues.dump

docker compose exec -t replica psql -U bda -d albergues -c "SELECT * FROM albergues WHERE camino = 'Portugués';"

```

```text
 id |       nome        | concello  |  camino   | prazas | prezo_noite 
----+-------------------+-----------+-----------+--------+-------------
  5 | Albergue Miñor    | Tui       | Portugués |     30 |       10.00
  6 | Albergue Ría Viva | Redondela | Portugués |     22 |       12.00
(2 rows)

```

> Recuperáronse os datos que tiña a táboa antes do borrado gracias á copia de seguridade.

---

## Parte C: Cae o primario

### 1. Saídas dos comandos

**Parar o primario:**

```bash
docker compose stop primario

```

```text
[+] stop 1/1
 ✔ Container e2-replicacion-primario-1 Stopped 

```

**Comprobar lectura na réplica:**

```bash
docker compose exec replica psql -U bda -d albergues -c "SELECT * FROM albergues;"

```

```text
 id |          nome          |   concello   |  camino   | prazas | prezo_noite 
----+------------------------+--------------+-----------+--------+-------------
  1 | Albergue Pedra Miúda   | Sarria       | Francés   |     32 |       12.00
  2 | Albergue O Fiadeiro    | Portomarín   | Francés   |     24 |       13.50
  3 | Albergue Casa da Ponte | Palas de Rei | Francés   |     40 |       11.00
  4 | Albergue Vieiras       | Melide       | Francés   |     28 |       14.00
  5 | Albergue Miñor         | Tui          | Portugués |     30 |       10.00
  6 | Albergue Ría Viva      | Redondela    | Portugués |     22 |       12.00
(6 rows)

```

**Intentar escribir na réplica:**

```bash
docker compose exec replica psql -U bda -d albergues -c "INSERT INTO albergues (nome, concello, camino, prazas, prezo_noite) VALUES ('Albergue Provisorio', 'Padrón', 'Portugués', 15, 10.00);"

```

```text
ERROR:  cannot execute INSERT in a read-only transaction

```

**Promover a réplica:**

```bash
docker compose exec replica psql -U bda -d albergues -c "SELECT pg_promote();"

```

```text
 pg_promote 
------------
 t
(1 row)

```

**Comprobar estado de recuperacion:**

```bash
docker compose exec replica psql -U bda -d albergues -c "SELECT pg_is_in_recovery();"

```

```text
 pg_is_in_recovery 
-------------------
 f
(1 row)

```

**Escribir na réplica promovida:**

```bash
docker compose exec replica psql -U bda -d albergues -c "INSERT INTO albergues (nome, concello, camino, prazas, prezo_noite) VALUES ('Albergue Promovido', 'Padrón', 'Portugués', 15, 10.00);"

```

```text
INSERT 0 1

```

**Arrancar o antigo primario e xerar diverxencia:**

```bash
docker compose start primario

```

```text
[+] start 1/1
 ✔ Container e2-replicacion-primario-1 Started

```

```bash
docker compose exec primario psql -U bda -d albergues -c "INSERT INTO albergues (nome, concello, camino, prazas, prezo_noite) VALUES ('Albergue Vello Primario', 'Santiago', 'Francés', 50, 20.00);"

```

```text
INSERT 0 1

```

```bash
docker compose exec replica psql -U bda -d albergues -c "INSERT INTO albergues (nome, concello, camino, prazas, prezo_noite) VALUES ('Albergue Nova Replica', 'Fisterra', 'Fisterra', 30, 15.00);"

```

```text
INSERT 0 1

```

### 2. Respostas ás preguntas da Parte C

**Que problema hai agora que os dous servidores aceptan escrituras?**

> Que os dous servidores funcionan de forma independente como primarios e aceptan modificadores de datos. Isto xera que os datos de ambos servidores se desincronizan e xa non poden unificarse de forma automática sen conflitos de claves primarias ou perdas de información.

**Que tería que pasar para que o antigo primario volvese ser unha réplica do novo?**

> Necesitaríase resincronizar o antigo primario para que recupere o novo tempo da réplica promovida.

**Durante canto tempo non se puideron facer reservas? De que dependeu?**

> O tempo de inactividade foi o intervalo entre o momento no que parou o servidor primario (Paso 1) e o momento no que se executou `SELECT pg_promote();` na réplica (Paso 4).
> De que dependeu: Nesa práctica, dependeu do tempo que tardei en executar os comandos de promoción manuais.

---

## Parte D: Preguntas finais

### 1. A replicación é asíncrona. Que dato poderiamos perder se o primario cae xusto despois de confirmar unha reserva?

Poderíanse perder as reservas que foron confirmadas no primario porque os rexistros aínda non se transmitiron ou non se aplicaron na replica no momento exacto do fallo físico.

### 2. Por que necesitamos copias de seguridade e réplicas? Pon un exemplo de problema que só resolve cada unha.

* **Réplica (Alta Dispoñibilidade):** Protexe contra fallos de hardware ou infraestrutura (un disco roto ou un servidor que apaga). *Exemplo:* Se o servidor *A* queima a súa placa base, a réplica asume o servizo en segundos sen perder a continuidade do negocio.
* **Copia de seguridade (Integridade e Histórico):** Protexe contra corrupción de datos ou erros humanos/lóxicos. *Exemplo:* Se un programador executa por erro `DELETE FROM reservas;`, a réplica borrará todas as reservas ao instante. A única forma de recuperar eses datos é restaurando un `pg_dump`.

### 3. Que fai a opción `-R` de `pg_basebackup` no script da réplica?

A opción `-R` indica a `pg_basebackup` que xere automaticamente a configuración necesaria para que a base de datos arranque en modo réplica. Xera o ficheiro `standby.signal` e garda a conexión ao primario, as seguintes veces xa arranca directamente co que ten.

### 4. Por que o usuario replicador necesita unha liña específica en `pg_hba.conf`?

En PostgreSQL, as conexións de replicación non se tratan como conexións a bases de datos normais, senón como un protocolo especial. Necesítase a liña para darlle permiso ao usuario replicador de autenticarse e transmitir o fluxo desde ese tipo de conexión especial.