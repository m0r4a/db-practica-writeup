Este es mi write up de cómo instalo, configuro, y administro las DBs `PostgreSQL` y `MySQL`

Para ambas pruebas se utilizó Ubuntu Server y su [documentación oficial](https://documentation.ubuntu.com/server/how-to/databases/install-postgresql/)

# Tabla de Contenidos

# Tabla de Contenidos

- [PostgreSQL](#postgresql)
 - [Instalación](#instalación)
 - [Configuración](#configuración)
   - [Permitiendo conexiones externas](#permitiendo-conexiones-externas)
   - [Configurando el usuario `postgres`](#configurando-el-usuario-postgres)
   - [Arreglando que el puerto no se está abriendo](#arreglando-que-el-puerto-no-se-está-abriendo)
   - [Conectándome a la db](#conectándome-a-la-db)
   - [Creando una DB y usuarios](#creando-una-db-y-usuarios)
   - [Configuración de memoria y recursos](#configuración-de-memoria-y-recursos)
   - [Logging para monitoreo](#logging-para-monitoreo)
   - [Configuración de Write-Ahead Log (WAL)](#configuración-de-write-ahead-log-wal)
   - [Script de backups](#script-de-backups)

# PostgreSQL

## Instalación

Bastante básico, simplemente

```bash
sudo apt install postgresql
```

## Configuración

### Permitiendo conexiones externas

1. En mi caso estoy creando la DB en una VM así que dentro de `/etc/postgresql/*/main/postgresql.conf` voy a agregar la IP de la interfaz puente entre la VM y mi computadora.

```
listen_addresses = 'localhost, 192.168.100.195'
```

2. Y reinicio el servicio.

```bash
sudo systemctl restart postgresql
```

### Configurando el usuario `postgres`

1. Conectarse a la base de datos `template1` como el usuario postgres.

```bash
sudo -u postgres psql template1
```

2. Ahora hay que cambiar la contraseña del usuario

```sql
ALTER USER postgres with encrypted password 'la_contraseña';
```

3. Después hay que configurar el archivo `/etc/postgresql/*/main/pg_hba.conf`

```
hostssl       template1  postgres  192.168.100.1/24  scram-sha-256  [OPTIONS]
```

> Esto tiene un error, el [OPTIONS] no debería estar pero esto se explica en la siguiente sección.

4. Reiniciar el servicio:

```bash
sudo systemctl restart postgresql.service
```

> Aquí le agregué el `.service` se puede usar de forma indistinta a menos de que haya algo como postgresql.socket por ejemplo


### Arreglando que el puerto no se está abriendo

Al intentar conectarme desde mi computadora a la db me dió error, fui a la VM que la tiene y vi que el puerto no se abría aunque reiniciara el servicio, lo verifiqué con:

```bash
ss -lntp | grep 5432
```

1. Entonces revisé el estado del servicio

```bash
sudo systemctl status postgresql
```

Y dice que está `Active (exited)` lo que significa que ejecutó un script y salió así que no es el servicio real que se ejecuta en segundo plano 

2. Me puse a revisar los logs del servicio postgresql

```bash
journalctl -u postgresql -e
```

Pero no me dió nada importante:

```
Sep 03 01:38:25 db-server systemd[1]: Starting postgresql.service - PostgreSQL RDBMS...
Sep 03 01:38:25 db-server systemd[1]: Finished postgresql.service - PostgreSQL RDBMS.
Sep 03 01:40:55 db-server systemd[1]: postgresql.service: Deactivated successfully.
Sep 03 01:40:55 db-server systemd[1]: Stopped postgresql.service - PostgreSQL RDBMS.
Sep 03 01:40:55 db-server systemd[1]: Stopping postgresql.service - PostgreSQL RDBMS...
Sep 03 01:40:55 db-server systemd[1]: Starting postgresql.service - PostgreSQL RDBMS...
Sep 03 01:40:55 db-server systemd[1]: Finished postgresql.service - PostgreSQL RDBMS.
```

3. Así que decidí revisar directamente el archivo del servicio a ver si había información

```bash
sudo systemctl cat postgresql
```

Esto es lo más relevante:

```
# The unit actually managing PostgreSQL clusters is postgresql@.service,
# instantiated as postgresql@15-main.service for individual clusters.
```

Revisé `postgresql@` y no hay nada, revisé el `postgresql@15-main` por probar y pues no había nada, ya suponía que revisar el `16-main`.

4. Al fin tengo información, esto fué lo que me dió el comando:

```bash
sudo systemctl status postgresql@16-main
```

Lo importante es esto:

```
CONTEXT:  line 19 of configuration file "/etc/postgresql/16/main/pg_hba.conf"
```

5. Conclusión

Al configurar el usuario de postgres en el punto 3 debí haber quitado el `[OPTIONS]`

### Conectándome a la db

```bash
psql --host 192.168.100.195 --username postgres --password --dbname template1
```

<p align="center">
    <img src="resources/conexion_db.png" alt="Imagen de mi pc conectándose a la db"/>
</p>

### Creando una DB y usuarios

1. Crear los usuarios

```sql
CREATE ROLE app_user WITH LOGIN PASSWORD 'una_contraseña';
CREATE ROLE readonly_user WITH LOGIN PASSWORD 'secure_password';
```

2. Crear la base de datos

```sql
CREATE DATABASE app_db OWNER app_user;
```

3. Configurar los permisos

```sql
GRANT CONNECT ON DATABASE app_db TO readonly_user;  -- Dar permiso de conectarse
GRANT USAGE ON SCHEMA public TO readonly_user;  -- Dar permiso para ver el esquema "public"
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;  -- Permitir al usuario hacer consultas SELECT en el esquema público
```

y con esto tenemos al usuario `app_user` que es dueño de la db `app_db` y al usuario `readonly_user` que tiene permiso de leer el esquema `public` de la db de `app_user`.

### Configuración de memoria y recursos

en `/etc/postgres/*/main/postgresql.conf`

```
# La VM que tiene la DB tiene 4GB de RAM
shared_buffers = '1GB'          # 25% de RAM total
effective_cache_size = '3GB'      # 50-75% de RAM total
work_mem = '8MB'
maintenance_work_mem = '256MB'
```

### Logging para monitoreo

```
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB
log_line_prefix = '%m [%p] %q%u@%d '
log_min_duration_statement = 1000 
log_statement = 'ddl'
#log_statement = 'all'           # Solo para desarrollo
```

### Configuración de Write-AHead Log (WAL)

WAL es un sistema de registro que ayuda con la durabilidad y consistencia de los datos. Al haber un cambio en la base de datos esto se escribe en un fichero WAL antes de realmente aplicarse.

1. Crear los directorios:

```
sudo mkdir -p /var/lib/postgresql/archive
sudo chown postgres:postgres /var/lib/postgresql/archive
```

2. En `/etc/postgresql/*/main/postgresql.conf`

```
wal_level = replica
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/archive/%f'
max_wal_senders = 0
```

- `wal_level`
    - `minimal`: solo contiene lo necesario para recuperarse tras un crash. No permite PITR ni replicación.
    - `replica`: permite PITR y replicación. Es el más común.
    - `logical`: tiene más detalle y con este puedes hacer **replicación lógica** (como replicar solo ciertas tablas).
- `archive_mode`
    - `off`:  No archiva WAL. Solo crash recovery.
    - `on`: Archiva WAL al **cerrar** segmentos (este es el estandar).
    - `always`: Archiva incluso segmentos usados por réplica en streaming. Útil si se necesita **garantizar copias** sin depender de réplicas.
- `archive_command`:
    - Este es el comando que se ejecuta cuando un segmento WAL está listo para archivarse.
    - También se puede usar `rsync -a %p <ruta>/%f` en lugar de sólo cp.
- `max_wal_senders`:
    - Es el número máximo de procesos que pueden enviar WAL a réplicas.
    - Solo es importante si hay servidores secundarios (standby).

Entonces:

- Si solo quieres **PITR sin réplicas** se usa:
    - `wal_level = replica`
    - `archive_mode = on`
    - `archive_command = (comando que copie a un disco o almacenamiento)`

- Si se necesita **replicación lógica** entonces:
    - `wal_level = logical`

- Si solo se requiere un **crash recovery básico**:
    - `wal_level = minimal`
    - `archive_mode = off`

### Script de backups

```bash
#!/bin/bash
# /usr/local/bin/pg_backup.sh

# Si cualquier comando falla todo el script falla
set -euo pipefail

# Configuración
BACKUP_DIR="/var/backups/postgresql"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7
LOG_FILE="/var/log/pg_backup.log"
DB_USER="postgres"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Crear los directorios necesarios si no existen
mkdir -p "$(dirname "$LOG_FILE")"
mkdir -p "$BACKUP_DIR"

log "Iniciando el backup de PostgreSQL"

# Hacer el backup con compresión
BACKUP_FILE="$BACKUP_DIR/full_backup_$DATE.sql.gz"

if pg_dumpall -U "$DB_USER" | gzip > "$BACKUP_FILE"; then

    chmod 600 "$BACKUP_FILE"

    # verificar se el archivo se creó y tiene algo
    if [[ -s "$BACKUP_FILE" ]]; then
        BACKUP_SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
        log "Backup completada de forma exitosa: $BACKUP_FILE ($BACKUP_SIZE)"
        
        # Limpiar backups antiguos
        DELETED=$(find "$BACKUP_DIR" -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete -print | wc -l)
        log "Copias de seguridad viejas $DELETED eliminadas"
    else
        log "ERROR: El archivo de la copia de seguridad está vacío o no fue creado"
        exit 1
    fi
else
    log "ERROR: Falló el comando pg_dumpall"
    exit 1
fi

log "Proceso de backup completado"
```

#### Consideraciones del script:

1. Los permisos del script deben ser los siguientes:

```
chmod 700 /usr/local/bin/pg_backup.sh
chown postgres:postgres /usr/local/bin/pg_backup.sh
```

Ya que si se vulnera un script que esté en el crontab se pueden ejecutar cosas en nombre del usuario que ejecute ese job.

2. Se debe configurar un método de configuración, ya sea un `.pgpass`

```bash
# Crear el archivo y su contenido
sudo mkdir -p /etc/postgresql/auth
sudo cat > /etc/postgresql/auth/.pgpass << 'EOF'
localhost:5432:db_seleccionada:postgres:contraseña
EOF

sudo chmod 600 /etc/postgresql/auth/.pgpass
sudo chown postgres:postgres /etc/postgresql/auth/.pgpass

# Configurar archivo de password en el .bashrc del usuario postgres
export PGPASSFILE="/etc/postgresql/auth/.pgpass"
```

O modifical el `pg_hba.conf` para permitir la autentacación por usuario del sistema:

```
local all postgres peer
```

3. Este script está pensado para ser ejecutado por el usuario `postgres`

```bash
sudo -u postgres crontab -e
```

Y la entry es la siguiente:

```
# Backups lunes, miércoles y viernes
0 2 * * 1,3,5 /usr/local/bin/pg_backup.sh
```

4. Esto puede ser más completo, enviar correos si el backup falla y demás pero no quiero hacer que este README tenga 1300 líneas.
