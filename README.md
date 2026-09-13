# DevOps MediaWiki Lab

Homelab project for learning DevOps practices and tools.

## Goal

Build a reproducible MediaWiki environment to practice Docker, Docker Compose,
Linux administration, Git workflows, persistent storage, secrets, backups,
monitoring and further CI/CD and Kubernetes practices.

## Architecture

The current stack consists of:

- MediaWiki 1.46
- MariaDB 11.4
- Docker Compose
- persistent Docker volumes for the database and uploaded files
- a MariaDB healthcheck
- a file-based Compose secret for the database password

See [docs/architecture.md](docs/architecture.md) for the planned target
architecture for later stages of the lab.

## Requirements

A Linux host with:

- Docker Engine
- Docker Compose plugin
- Git

Check:

```bash
docker --version
docker compose version
git --version
```

## Initial setup

Clone the repository and enter the project directory:
```bash
git clone https://github.com/gorokhov-byte/devops-mediawiki-lab.git
cd devops-mediawiki-lab
```
Create the local environment file:
```bash
cp .env.example .env
chmod 600 .env
```
Create the directory for local secrets:
```bash
mkdir -p secrets
chmod 700 secrets
```
Create the database password file without putting the password into shell history:
```bash
read -rsp "DB password: " DBPASS
echo
printf '%s' "$DBPASS" > secrets/db_password.txt
unset DBPASS

chmod 644 secrets/db_password.txt
```
The password file is intentionally excluded from Git.

## First start

On the first start, use only the base Compose file:
```bash
sudo docker compose up -d
```
Check the services:
```bash
sudo docker compose ps
```
MariaDB should eventually report:

Up ... (healthy)

Open MediaWiki in a browser.

If the browser runs on the Docker host:

```text
http://localhost:8080
```

If Docker runs on a remote VM, open:

```text
http://<docker-host-ip>:8080
```

When the installer asks for database settings, use:

Database type: MariaDB / MySQL
Database host: db
Database name: mediawiki
Database user: wiki
Database password: the value stored in secrets/db_password.txt

The database host is db because Docker Compose provides internal DNS between
services on the project network.

Complete the MediaWiki installation and download the generated
LocalSettings.php.

Copy `LocalSettings.php` into the repository root on the Docker host.

Set permissions so that the web server inside the container can read the file:

```bash
chmod 644 LocalSettings.php
```

Edit `LocalSettings.php` and replace the generated database password line:

```php
$wgDBpassword = "your-password";
```

with:

```php
$wgDBpassword = trim( file_get_contents( '/run/secrets/db_password' ) );
```

This keeps the database password out of `LocalSettings.php`. The same Compose
secret is mounted into both the MariaDB and MediaWiki services.

The file must remain outside Git because it contains sensitive installation
settings.

## Running the installed wiki

After `LocalSettings.php` exists, start the stack using both Compose files:
```bash
sudo docker compose \
  -f compose.yaml \
  -f compose.installed.yaml \
  up -d
```
The second Compose file adds the bind mount:

./LocalSettings.php -> /var/www/html/LocalSettings.php

Check the stack:
```bash
sudo docker compose \
  -f compose.yaml \
  -f compose.installed.yaml \
  ps
```
## Useful commands

View service logs:
```bash
sudo docker compose logs db
sudo docker compose logs wiki
```
Follow logs:
```bash
sudo docker compose logs -f wiki
```
Open a shell inside the MediaWiki container:
```bash
sudo docker compose exec wiki sh
```
Check that the MediaWiki container can resolve the database service:
```bash
sudo docker compose exec wiki getent hosts db
```
Check the MariaDB health status:
```bash
sudo docker inspect \
  --format='Status={{.State.Health.Status}}' \
  "$(sudo docker compose ps -q db)"
```
## Database backup

This procedure backs up only the MediaWiki database. Uploaded files stored in
the `images` volume require a separate backup.

Create a local backup directory:
```bash
mkdir -p backups
chmod 700 backups
```
Set a timestamped backup filename:
```bash
BACKUP="backups/mediawiki-$(date +%F-%H%M).sql"
```
Create a logical MariaDB dump:
```bash
sudo docker compose exec -T db sh -c \
'mariadb-dump \
  -u "$MARIADB_USER" \
  --password="$(cat /run/secrets/db_password)" \
  "$MARIADB_DATABASE"' \
> "$BACKUP"

chmod 600 "$BACKUP"
```
Verify that the file was created:
```bash
ls -lh "$BACKUP"
```
## Database restore

This procedure restores only the MediaWiki database.

Stop MediaWiki before restoring the database:
```bash
sudo docker compose stop wiki
```
Restore the dump:
```bash
sudo docker compose exec -T db sh -c \
'mariadb \
  -u "$MARIADB_USER" \
  --password="$(cat /run/secrets/db_password)" \
  "$MARIADB_DATABASE"' \
< "$BACKUP"
```
Start MediaWiki again:
```bash
sudo docker compose start wiki
```
## Persistent data

MariaDB data is stored in a Docker volume mounted at:

/var/lib/mysql

Uploaded MediaWiki files are stored in another Docker volume mounted at:

/var/www/html/images

Deleting and recreating containers therefore does not delete persistent
application data.

A Docker volume is not a backup. Backups must be stored separately and restore
procedures should be tested.

## Local files excluded from Git

The repository does not track:

.env
LocalSettings.php
secrets/
backups/

Never commit passwords, generated MediaWiki secrets or database backups.
