# oracle19c-docker
A lightweight and configurable Oracle 19c docker image.

Oracle has introduced the concept of container databases (CDB) and pluggable databases (PDB). Containers are used for multi-tenancy and contain pluggable databases. Pluggable databases are what you are probably used to, a self contained database that you connect to. By default, this image creates one CDB, and one PDB within that CDB.

Note that Oracle is shifting away from an SID and using service names instead. PDB use service names, CDB use SIDs. However, usernames for CDB must start with C##, ie C##STEVE. So to make things simple, we are focusing on just using the single PDB.

Before you begin
----------------

1. Clone `https://github.com/oracle/docker-images`.
1. Download the Oracle Database 19c binary from [http://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html](https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html)
    1. For X86, get `LINUX.X64_193000_db_home.zip`.
      ![image](https://github.com/user-attachments/assets/e534d122-9d7a-4c65-a17e-1978667180ca)

1. Put the zip in the `OracleDatabase/SingleInstance/dockerfiles/19.3.0` directory. **Do not unzip it.**
   ![image](https://github.com/user-attachments/assets/5f175b19-2f5c-4a9e-bf16-68a5ac7ececd)

1. In your docker application (e.g. Colima, Docker Desktop etc), ensure you have a large enough amount of memory allocated to docker. These instructions will set the total memory to 4000MB, so make sure Docker has a value higher than that.

Start your container environment
--------------------------------
These instructions assume you are using Docker desktop. Start up docker by opening Docker desktop app.
- For windows: [https://docs.docker.com/desktop/install/windows-install](https://docs.docker.com/desktop/install/windows-install/)
- For Linux: [https://docs.docker.com/desktop/install/ubuntu](https://docs.docker.com/desktop/install/ubuntu/)
- For MacOs: [https://docs.docker.com/desktop/install/mac-install](https://docs.docker.com/desktop/install/mac-install/)

Building
--------
For Unix (MacOS) and Linux, you can use bash terminal, and for Windows, you have to use **git bash** (It is installed automatically when you install git in your machine).
````
cd OracleDatabase/SingleInstance/dockerfiles
./buildContainerImage.sh -v 19.3.0 -e
````

If the build fails saying you are out of space, check how much space you have available on your disk. If it looks ok, prune old Docker images via: 
`yes | docker image prune > /dev/null`

Running
-------

To use the sensible defaults (Run this command in git bash in windows):
```bash
docker run \
--name oracle19c --ulimit nofile=65536:65536 \
-p 1521:1521 \
-p 5500:5500 \
-e ORACLE_PDB=orcl \
-e ORACLE_PWD=password \
-e INIT_SGA_SIZE=3000 \
-e INIT_PGA_SIZE=1000 \
-v /opt/oracle/oradata \
-d \
oracle/database:19.3.0-ee
```

On first run, the database will be created and setup for you. This will take about 10-15 minutes. Open the container logs and watch the progress. Then you can connect.

Optionally, you can use the following run command to avoid getting "No disk space" issues as you gradually insert more and more data into your database.

```bash
docker run \
--name oracle19c --ulimit nofile=65536:65536 \
-p 1521:1521 -p 5500:5500 \
-e ORACLE_PDB=orcl \
-e ORACLE_PWD=password \
-e INIT_SGA_SIZE=3000 \
-e INIT_PGA_SIZE=1000 \
-v /Users/<your-username>/path/to/store/db/files/:/opt/oracle/oradata \
-d \
oracle/database:19.3.0-ee
```

Configuration
-------------

```
   --name:        The name of the container (default: auto generated)
   -p:            The port mapping of the host port to the container port.
                  Two ports are exposed: 1521 (Oracle Listener), 5500 (OEM Express)
   -e ORACLE_SID: The Oracle Database SID that should be used (default: ORCLCDB)
   -e ORACLE_PDB: The Oracle Database Service Name that should be used (default: ORCLPDB1)
   -e ORACLE_PWD: The Oracle Database SYS password (default: auto generated)
   -e INIT_SGA_SIZE: The amount of SGA to allocate to Oracle. This should be 75% of the total memory you want Oracle to use. 
   -e INIT_PGA_SIZE: The amount of PGA to alloxate to oracle. This should be the remaining 25% of the total memory you want Oracle to use.  
   -e ORACLE_CHARACTERSET: The character set to use when creating the database (default: AL32UTF8)
   -v /opt/oracle/oradata
                  The data volume to use for the database.
                  Has to be writable by the Unix "oracle" (uid: 54321) user inside the container!
                  If omitted the database will not be persisted over container recreation.
   -v /Users/<your-username>/path/to/store/db/files/:/opt/oracle/oradata
                  Mount the data volume into one of your local folders.
                  If omitted you might run into a "No disk space" issue at some point as your database keeps growing and docker does not resize its volume.
   -d:            Run in detached mode. You want this otherwise `Ctrl-C` will kill the container.
```

Note that if you do not specify INIT_SGA_SIZE and INIT_PGA_SIZE then Oracle will determine the memory to allocate based on the number of CPUs, available memory in your machine etc, and for desktop environments this could be from about 2000MB to 5000MB. If you want control over this, set the values.

Connecting to Oracle
--------------------
You have to make sure that your container is ready to use before connecting to it using SQL Developer or your preferred IDE. You have see the logs of the container using this docker command.

```bash
docker logs --follow oracle19c                                                                                                                                            at 03:29:33 PM
```
When you are running it for the first time, it gonna take nearly 10 min to complete the initialisation process. You have to verify whether the database is ready to use or not by watching container logs (Containers means your database in this case). 
![image](https://github.com/user-attachments/assets/4ee3b597-7bbf-41b5-bf0f-969311d60203)

Once the container has been started you can connect to it like any other database. Note we are using `Service Name` and not the SID (since PDB uses Service Name).

For example, SQL Developer:
```
Hostname: localhost
Port: 1521
Service Name: <your service name>
Username: sys
Password: <your password>
Role: AS SYSDBA

```
![image](https://github.com/user-attachments/assets/64b74ae1-c50d-42cf-a5f6-d3d1804ca18b)
![image](https://github.com/user-attachments/assets/523f6a5e-04e0-40a2-be3d-1346317b181f)

Changing the password
---------------------

Note: If you did not set the `ORACLE_PWD` parameter, check the docker run output for the password.

The password for the SYS account can be changed via the `docker exec` command. Note, the container has to be running:

First run `docker ps` to get the container ID. Then run:
`docker exec <container id> ./setPassword.sh <new password>`

Getting a shell on the container
--------------------------------
First run `docker ps` to get the container ID. Then run:
`docker exec -it <container id> /bin/bash`

Or as root:
`docker exec -u 0 -it <container id> /bin/bash`

Using SQLPlus within the container
----------------------------------

Once on the container (see above), connect to SQLPlus like so:
```
sqlplus /nolog
connect sys/password@orcl as sysdba
```
