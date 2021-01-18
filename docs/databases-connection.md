# Databases connection

## :speech_balloon: Description

To explore the postgresql databases, you can use a *free multi-platform database tool* like :

- [`DBeaver`](https://dbeaver.io/download/ "Download page")

or continue with docker philosophy and run a :

- [`pgAdmin`](https://www.pgadmin.org/docs/pgadmin4/development/container_deployment.html) container.

## pgAdmin container

```bash
docker run -p 8085:80 \
    --net=kafka-connect-pgsql_default \
    --name pgadmin4 \
    -e PGADMIN_DEFAULT_EMAIL=user@domain.com \
    -e PGADMIN_DEFAULT_PASSWORD=SuperSecret \
    -d \
    dpage/pgadmin4
```

### Login

On address <http://localhost:8085/>, fill the login information with :

- Email Address / Username : `user@domain.com`
- Password : SuperSecret

### Source database connection

- Add a new server _(Dashboard/Add New Server)_ :
  - **General tab**
    - Name : mySourceDatabase
  - **Connection tab**
    - Hostname/address : postgres-source
    - Port : 5432
    - Maintenance database : db
    - Username : user
    - Password : password

### Sink database connection

- Add a new server _(Dashboard/Add New Server)_ :
  - **General tab**
    - Name : mySinkDatabase
  - **Connection tab**
    - Hostname/address : postgres-sink
    - Port : 5432
    - Maintenance database : db
    - Username : user
    - Password : password

> See also the [pgAdmin documentation](https://www.pgadmin.org/docs/pgadmin4/latest/server_dialog.html)

## DBeaver

In a `DBeaver` installed on host machine, the connection informations are a bit different :

### Source database connection

- Hostname/address : localhost
- :warning: Port : 5432
- Maintenance database : db
- Username : user
- Password : password

### Sink database connection

- Hostname/address : localhost
- :warning: Port : 5433
- Maintenance database : db
- Username : user
- Password : password
