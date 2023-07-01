---
title: "Dockerizing Oracle Enterprise Edition"
date: 2022-04-06T14:44:55+03:00
draft: false
---

## Docker image for Oracle Enterprise Edition

Not suprisingly, first you need to get a docker image for Oracle Enterprise Edition. Since this version is a paid version, you should get the image from some private repository. It could be your organization’s private Docker repository etc. For instance in my case, I obtain the image for Oracle Enterprise Edition 12.2.0.1 from my organization’s Azure Docker Registry.

## Running docker container

After getting image properly now we can run the container. But before getting into this, you’ll probably want to initialize your database with some SQL scripts, won’t you?
For this purpose, we need to place our scripts in folder **/opt/oracle/scripts/setup** or **/opt/oracle/scripts/startup** in container. You can mount script folder to one of them.

```
/opt/oracle/scripts/setup -> Scripts will be run only during the initial database creation.
/opt/oracle/scripts/startup -> Scripts will be run after the initial database creation is complete.
/docker-entrypoint-initdb.d -> Symbolic link that references /opt/oracle/scripts, you can also put scripts by using this symbolic link
```

You can name your scripts with prefix to specift execution order like following:

```
01_create_tablespaces.sql
02_create_user.sql
.
.
.
```

**Note:** You might need to add the following line on top of your all scripts:

```
ALTER SESSION SET CONTAINER=PDB_NAME;
```

To avoid data loss upon container restart, we can mount a directory to /opt/oracle/oradata folder inside container. If you don’t do this, actually you’ll also end up with “No space left” error especially if you create tablespace in your scripts. To avoid permission denied errors, you should change owner of the mounted directory on the host.

```
chown 54321:54322 /path/to/mounted/folder

#54321 is the uid of oracle user inside the container
#54322 is the gid of the dba group inside the container
```

Full docker run command to start a container:

```
docker run -d --name myoracledb -p 1521:1521 -p 5500:5500 \
-v /path/to/scripts:/opt/oracle/scripts/startup \
-v /path/to/data:/opt/oracle/oradata \
-e ORACLE_SID=orcl -e ORACLE_PDB=ORCLPDB -e ORACLE_PWD=password \
cool_name_of_image:12.2.0.1
```

You can follow the logs during the creation of container:

```
docker logs -f myoracledb
```

#### References

1. https://docs.oracle.com/en/database/oracle/oracle-database/21/deeck/oracle-database-enterprise-edition-installation-guide-docker-containers-oracle-linux.pdf