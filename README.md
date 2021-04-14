# Setup Graph Server using Docker

The docker image build files, sample datasets, and use case exmaples, for Oracle Property Graph.

Architecture:

![](https://user-images.githubusercontent.com/4862919/80330080-632e9a00-886e-11ea-822e-0a96e40dbbf9.jpg)

Oracle Database is required before setting up Graph Server because its authentication is based on the database users. For standalone usage, please see [Standalone mode](#Standalone_mode) section.

- [Setup Database](#Setup_Database)
- [Setup Graph Server](#Setup_Graph_Server)

# Setup Database

## Option 1. Use your existing database

If you have an environment running Oracle Database (>= 12.2), the new Graph Server container can integrate with it. Please go to the next step to configure the database.

## Option 2. Use Container Registry image (19c EE)

## Option 3. Build image by yourself (18c XE)

Clone `docker-images` repository.

    $ cd <your-work-directory>
    $ git clone https://github.com/oracle/docker-images.git

Download Oracle Database.

* [Oracle Database 19.3.0 for Linux x86-64 (ZIP)](https://www.oracle.com/database/technologies/oracle-database-software-downloads.html)

Put `LINUX.X64_193000_db_home.zip` under:
* `docker-images/OracleDatabase/SingleInstance/dockerfiles/19.3.0/`

Build the image.

    $ cd docker-images/OracleDatabase/SingleInstance/dockerfiles/
    $ bash buildDockerImage.sh -v 19.3.0 -e

Start the container. This step takes time for the first time.

    $ cd oracle-pg/
    $ docker-compose up database
    ...
    database_1      | Completing Database Creation
    ...
    database_1      | #########################
    database_1      | DATABASE IS READY TO USE!
    database_1      | #########################

You need to start the container if it is stopped.

    $ docker start database

You will get this error when you try to connect before the database is created.

    $ docker exec -it database sqlplus sys/Welcome1@localhost:1521/orclpdb1 as sysdba
    ...
    ORA-12514: TNS:listener does not currently know of service requested in connect

To check the progress, see logs.

    $ docker logs -f database

## Configure Database

You need to apply the PL/SQL patch to the database.

Go to the following pages and download the packages.

* [Oracle Graph Server and Client 21.1](https://www.oracle.com/database/technologies/spatialandgraph/property-graph-features/graph-server-and-client/graph-server-and-client-downloads.html)

- oracle-graph-plsql-21.1.0.zip

Connect to the database container.

```
$ docker exec -it database sqlplus sys/WELcome123##@orclpdb1 as sysdba
```

Configure Property Graph features. This script was extracted from oracle-graph-plsql-xx.x.x.zip.

```
SQL> @/home/oracle/scripts/oracle-graph-plsql/19c_and_above/opgremov.sql
SQL> @/home/oracle/scripts/oracle-graph-plsql/19c_and_above/catopg.sql
```

# Setup Graph Server

## Clone this Git Repository

    $ cd <your-work-directory>
    $ git clone https://github.com/ryotayamanaka/setup_pg_docker.git

## Download and Extract Packages for Graph Server and Client

Go to the following pages and download the packages.

* [Oracle Graph Server and Client 21.1](https://www.oracle.com/database/technologies/spatialandgraph/property-graph-features/graph-server-and-client/graph-server-and-client-downloads.html)
* [Oracle JDK 11](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html) (No cost for personal use and development use)

Put the following files under `packages/` directory.
 
- oracle-graph-21.1.0.x86_64.rpm
- oracle-graph-client-21.1.0.zip
- jdk-11.0.10_linux-x64_bin.rpm

## Start Container

Build the image.

```
docker build . \
--tag graph-server:21.1.0 \
--build-arg VERSION_GSC=<version of Graph Server and Client> \
--build-arg VERSION_JDK=<version of JDK>
```

Example:

```
docker build . \
--tag graph-server:21.1.0 \
--build-arg VERSION_GSC=21.1.0 \
--build-arg VERSION_JDK=11.0.10
```

Start a container.

```
docker run \
--name <container name> \
--publish <host port>:7007 \
--volume $PWD/data:/opt/oracle/graph/data \
--volume $PWD/pgx.conf:/etc/oracle/graph/pgx.conf \
graph-server:21.1.0
```

Example:

```
docker run \
--name graph-server \
--publish 7007:7007 \
--volume $PWD/data:/opt/oracle/graph/data \
--volume $PWD/pgx.conf:/etc/oracle/graph/pgx.conf \
graph-server:21.1.0
```

## Connect to Graph Server

```
docker exec -it graph-server /bin/bash
```

Access Graph Visualization using **FireFox**.

* Graph Visualization - https://localhost:7007/ui/ (User: graph_dev, Password: WELcome123##)





# Appendix: Setup Jupyter

* Jupyter - http://localhost:8888/

To stop, restart, or remove the containers, see [Appendix 2](#appendix-2).


`Cnt+C` to quit.

## Appendix 2

To start, stop, or restart the containers.

    $ docker-compose start|stop|restart

To remove the docker containers.

    $ cd oracle-pg/
    $ docker-compose down
