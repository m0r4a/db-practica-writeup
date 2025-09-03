Este es mi write up de cómo instalo, configuro, y administro las DBs `PostgreSQL` y `MySQL`

Para ambas pruebas se utilizó Ubuntu Server y su [documentación oficial](https://documentation.ubuntu.com/server/how-to/databases/install-postgresql/)

# PostgreSQL

## Instalación

Bastante básico, simplemente

```bash
sudo apt install postgresql
```

## Configuración

### Permitiendo conexiónes externas

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

Al intentar conectarme desde mi computadora a la db me dió error, fui a la VM que la tiene y vi que el puerto no se abría aunque reiniciara el servicio, lo verfiqué con:

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
