*Note: The scripts here are my own, not Oracle product.*

# Setup Graph Server using Docker

The docker image build files, sample datasets, and use case exmaples, for Oracle Property Graph.

Architecture:

![](https://user-images.githubusercontent.com/4862919/80330080-632e9a00-886e-11ea-822e-0a96e40dbbf9.jpg)

Oracle Database is required before setting up Graph Server because its authentication is based on the database users.

- [Setup Database](#Setup_Database)
- [Setup Graph Server](#Setup_Graph_Server)

# Setup Database

## Option 1. Use your existing database

If you have an environment running Oracle Database (>= 12.2), the new Graph Server container can integrate with it. Please go to the next step to configure the database.

## Option 2. Create a database container

Oracle Database [Express Edition (XE)](https://www.oracle.com/database/technologies/appdev/xe.html) is freely available, and we can get the scripts to build Docker image for XE 18c from the official GitHub repository.

Clone `docker-images` repository.

    $ cd ~/
    $ mkdir oracle
    $ cd oracle
    $ git clone https://github.com/oracle/docker-images.git

Build docker image. This step requires about 4GB memory.

    $ cd docker-images/OracleDatabase/SingleInstance/dockerfiles/18.4.0/
    $ docker build -t oracle/database:18.4.0-xe -f Dockerfile.xe .

Launch Oracle Database on a docker container.

    $ docker run --name database \
      -p 1521:1521 -e ORACLE_PWD=Welcome1 \
      -v $HOME:/host-home \
      oracle/database:18.4.0-xe

Once you got the message below, the database is ready.

    #########################
    DATABASE IS READY TO USE!
    #########################

Open another console and try connecting with SQL*Plus.

    $ docker exec -it database sqlplus sys/Welcome1@xepdb1 as sysdba

You will get this error when the database is not ready yet.

    ORA-12514: TNS:listener does not currently know of service requested in connect descriptor

You can stop the container (or quit with Ctl+C) and restart it.

    $ docker stop database
    $ docker start database

To check the progress, see logs.

    $ docker logs -f database

## Configure Database

You need to apply the PL/SQL patch to the database.

Go to the [Oracle Graph Server and Client](https://www.oracle.com/database/technologies/spatialandgraph/property-graph-features/graph-server-and-client/graph-server-and-client-downloads.html) page and download the PL/SQL package.

- oracle-graph-plsql-21.4.0.zip

Unzip the content under `oracle/oracle-graph-plsql/`.

    $ cd ~/oracle/
    $ unzip oracle-graph-plsql-21.4.0.zip -d oracle-graph-plsql

Connect to the database container.

    $ docker exec -it database sqlplus sys/Welcome1@xepdb1 as sysdba

Enable the graph feature. Please note `$HOME` of the host is mounted to `/host-home` in the container.

    SQL> @/host-home/oracle-graph/oracle-graph-plsql/18c_and_below/opgremov.sql
    SQL> @/host-home/oracle-graph/oracle-graph-plsql/18c_and_below/catopg.sql
    SQL> exit

## Create a User

Connect to the database container.

    $ docker exec -it database sqlplus sys/Welcome1@xepdb1 as sysdba

Create a database user `graphuser` and grant necessary privileges.

```sql
CREATE USER graphuser
IDENTIFIED BY Welcome1
DEFAULT TABLESPACE users
TEMPORARY TABLESPACE temp
QUOTA UNLIMITED ON users;

GRANT
  alter session 
, create procedure 
, create sequence 
, create session 
, create table 
, create trigger 
, create type 
, create view
, graph_developer -- This role is required for using Graph Server
TO graphuser;
```

Exit and try connecting as the new user.

    SQL> exit
    $ docker exec -it database sqlplus graphuser/Welcome1@xepdb1

## Create a Graph 

You need SQLcl + PGQL plugin to run PGQL queries. (SQL*Plus does not support PGQL.)

[This](https://github.com/ryotayamanaka/sqlcl-pgql) is the instruction to run SQLcl on a Docker container.

Once it is installed, connect to the database.

    $ sql graphuser/Welcome1@host.docker.internal:1521/xepdb1
    SQL>

Check if you can enable the PGQL mode.

    $ pgql auto on
    PGQL>

Run some PGQL queries.

```sql
CREATE PROPERTY GRAPH graph1;

INSERT INTO graph1 VERTEX v
LABELS (PERSON) PROPERTIES (v.id = 'p1', v.NAME = 'Alice');

INSERT INTO graph1 VERTEX v
LABELS (CAR) PROPERTIES (v.id = 'd1', v.BRAND = 'Toyota');

INSERT INTO graph1 EDGE e BETWEEN src AND dst
LABELS (HAS) PROPERTIES (e.SINCE = 2017)
FROM MATCH ( (src), (dst) ) ON graph1
WHERE src.id = 'p1' AND dst.id = 'd1';

COMMIT;

SELECT c.BRAND, h.SINCE
FROM MATCH (p)-[h:HAS]->(c) ON graph1
WHERE p.NAME = 'Alice';

DELETE v
FROM MATCH (v) ON graph1;

COMMIT;

DROP PROPERTY GRAPH graph1;
```

Exit from SQLcl.

    PGQL> exit

# Setup Graph Server

## Clone this Git Repository

    $ cd ~/oracle/
    $ git clone https://github.com/ryotayamanaka/setup_pg_docker.git

## Download and Extract Packages for Graph Server and Client

Go to the following pages and download the packages.

* [Oracle Graph Server and Client 21.4](https://www.oracle.com/database/technologies/spatialandgraph/property-graph-features/graph-server-and-client/graph-server-and-client-downloads.html)
* [Oracle JDK 11](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html) (No cost for personal use and development use)

Put the following files under `packages/` directory.
 
- oracle-graph-21.4.0.x86_64.rpm
- jdk-11.0.10_linux-x64_bin.rpm

## Start Container

Build the image.

```
docker build . \
--tag graph-server:<version of Graph Server and Client> \
--build-arg VERSION_GSC=<version of Graph Server and Client> \
--build-arg VERSION_JDK=<version of JDK>
```

Example:

```
docker build . \
--tag graph-server:21.4.0 \
--build-arg VERSION_GSC=21.4.0 \
--build-arg VERSION_JDK=11.0.10
```

Start a container.

```
docker run \
--name <container name> \
--publish <host port>:7007 \
--volume $PWD/pgx.conf:/etc/oracle/graph/pgx.conf \
graph-server:21.4.0
```

Example:

```
docker run \
--name graph-server \
--publish 7007:7007 \
--volume $PWD/pgx.conf:/etc/oracle/graph/pgx.conf \
graph-server:21.4.0
```

## Connect to Graph Server

```
docker exec -it graph-server /bin/bash
```

Access Graph Visualization using web browser.

* Graph Visualization - https://localhost:7007/ui/ (User: graphuser, Password: Welcome1)
