## Parte A

### Comandos

`docker compose up -d primario`

Crear a copia: `docker compose exec -t primario pg_dump -U bda -F c -f /var/lib/postgresql/data/albergues.dump albergues`

Comprobar que se fixo vendo si ocupa espacio: `docker compose exec primario ls -lh /var/lib/postgresql/data/albergues.dump`

Copiar fóra do contenedor: `docker compose cp primario:/var/lib/postgresql/data/albergues.dump ./albergues.dump`

*primario:...*: Indica o nome do contedor/servizo e a ruta interna onde está o arquivo.

*./albergues.dump*: Indica que se quere gardar na carpeta actual de WSL co mesmo nome.

Eliminar a táboa albergues: `docker compose exec -t primario psql -U bda -d albergues -c "DROP TABLE IF EXISTS albergues CASCADE;"`

Restaurar a copia: `docker compose exec -t primario pg_restore -U bda -d albergues -F c /var/lib/postgresql/data/albergues.dump`

Listar as táboas para comprobar que siguen aí: `docker compose exec -t primario psql -U bda -d albergues -c "\dt"`

Listar os albergues para comprobar que están todos: `docker compose exec -t primario psql -U bda -d albergues -c "SELECT * FROM albergues;"`

### Se fixeras a copia ás 9:00 e o desastre ocorrese ás 13:00, que datos se perderían?
Perderíanse todos os datos que se engadiran á táboa despois de facer a copia, é dicir, neste caso todos os datos posteriores ás 9:00.



## Parte B
### Saídas
#### ``docker compose logs -f replica``

```replica-1  | Réplica baleira: copiando os datos do primario...
replica-1  | waiting for checkpoint 
replica-1  | 30828/30828 kB (100%), 0/1 tablespace
replica-1  | 30828/30828 kB (100%), 1/1 tablespace
replica-1  | Copia completada. Arrancando como réplica.
replica-1  | 2026-10-22 16:06:04.349 UTC [1] LOG:  starting PostgreSQL 17.11 (Debian 17.11-1.pgdg13+2) on x86_64-pc-linux-gnu, compiled by gcc (Debian 14.2.0-19) 14.2.0, 64-bit
replica-1  | 2026-10-22 16:06:04.349 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
replica-1  | 2026-10-22 16:06:04.349 UTC [1] LOG:  listening on IPv6 address "::", port 5432
replica-1  | 2026-10-22 16:06:04.501 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
replica-1  | 2026-10-22 16:06:04.601 UTC [12] LOG:  database system was interrupted; last known up at 2026-10-22 16:06:03 UTC
replica-1  | 2026-10-22 16:06:04.812 UTC [12] LOG:  starting backup recovery with redo LSN 0/2000028, checkpoint LSN 0/2000080, on timeline ID 1
replica-1  | 2026-10-22 16:06:04.812 UTC [12] LOG:  entering standby mode
replica-1  | 2026-10-22 16:06:04.884 UTC [12] LOG:  redo starts at 0/2000028
replica-1  | 2026-10-22 16:06:04.924 UTC [12] LOG:  completed backup recovery with redo LSN 0/2000028 and end LSN 0/2000120
replica-1  | 2026-10-22 16:06:04.924 UTC [12] LOG:  consistent recovery state reached at 0/2000120
replica-1  | 2026-10-22 16:06:04.925 UTC [1] LOG:  database system is ready to accept read-only connections
replica-1  | 2026-10-22 16:06:04.996 UTC [13] LOG:  started streaming WAL from primary at 0/3000000 on timeline 1
```



### Explicación paso 5



## Parte C
### Que problema hai agora que os dous servidores aceptan escrituras?
#### Saída
#### Resposta

### Que tería que pasar para que o antigo primario volvese ser unha réplica do novo?
#### Saída
#### Resposta

### Durante canto tempo non se puideron facer reservas? De que dependeu?
#### Saída
#### Resposta



## Parte D
### A replicación é asíncrona. Que dato poderiamos perder se o primario cae xusto despois de confirmar unha reserva?

### Por que necesitamos copias de seguridade e réplicas? Pon un exemplo de problema que só resolve cada unha.

### Que fai a opción `-R` de `pg_basebackup` no script da réplica?

### Por que o usuario replicador necesita unha liña específica en `pg_hba.conf`?