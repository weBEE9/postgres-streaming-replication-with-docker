# PostgresQL Streaming Replication with Docker

Ref: [PostgreSQL Replication with Docker](https://medium.com/swlh/postgresql-replication-with-docker-c6a904becf77)

### Setup master postgres database

Config out master database in `docker-compose.yml`,

```yaml
version: '3'

services:
  database-master:
    image: postgres:14-alpine
    container_name: replication_master
    restart: always
    volumes:
      - ./data:/var/lib/postgresql/data
    ports:
      - "9876:5432"
    environment:
      - POSTGRES_USER=replication_test
      - POSTGRES_PASSWORD=replication_test
      - POSTGRES_DB=replication_test
```

and then start it.

```sh
$ docker-compose up -d replication_master
```

After master DB started, let's create a `ROLE` for replication usage,

```sql
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'replicator';
```

and create a physical replication slot.

```sql
SELECT * FROM pg_create_physical_replication_slot('replication_slot_slave1');
```

Finally edit the `pg_hba.conf` to allow role `replicator` access the database.

```conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
# IPv6 local connections:
host    all             all             ::1/128                 trust
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            trust
host    replication     all             ::1/128                 trust
# Allow replication connections for USER replicator
host    replication     replicator      0.0.0.0/0               trust

host all all all scram-sha-256
```

Restart the master database after `pg_hba.conf` edited.

```sh
$ docker-compose restart replication_master
```

## Backup the master database and restore it for the slave database

Use `pg_basebackup` command to do it.

```sh
$ pg_basebackup -D /tmp/postgresslave -S replication_slot_slave1 -X stream -P -U replicator -Fp -R
```

## Setup slave postgres database

Config out slave database in `docker-compose.yml`.

```yaml
version: '3'

services:
  database-slave:
    image: postgres:14-alpine
    container_name: postgres_slave
    restart: always
    volumes:
      - ./data-slave:/var/lib/postgresql/data
    ports:
      - "9877:5432"
```

Before start it, copy everything from the `/tmp/postgresslave` in the master database container and put it into `/data-slave`, which is the mount point for the slave database.

After coping, edit the `postgresql.auto.conf` so the slave database could read the correct information for connecting to the master database.

```conf
# can use 'replication_master' as host because they're in the same docker network
primary_conninfo = 'host=replication_master port=5432 user=replicator password=replicator'
primary_slot_name = 'replication_slot_slave1'
restore_command = 'cp /var/lib/postgresql/data/pg_wal/%f "%p"'
```

Now start the slave database and see if streaming replication works.

```sh
$ docker-compose up -d replication-slave
```

## Check if the slot is activated

```sql
SELECT * FROM pg_replication_slots;
```