---
title:  Creating Mysql databases in Docker
author: Will Andrews
date: 2018-10-09
order: 23
---

Recently I've been playing around with other databases other than Microsoft Sql Server. I've also been trying to use docker more, so to achieve both of these, I've been using new database images in Docker. 

## Prerequisites 

* Docker using Linux containers

* Mysql installed (Mysql workbench will help as well)


## Create docker container with Mysql image
```
docker run -p 3306:3306 -d --name {container name} -e MYSQL_ROOT_PASSWORD={password} mysql/mysql-server
```

## Check container is running
```
docker ps
```

## Connect to the Mysql server inside container and create a new user
```
docker exec -it {container name} bash

mysql -uroot -p{password}
```

(You'll now be running the Mysql cli)
```
create user '{username}'@'%' identified by '{password}';

grant all privileges on *.* to '{username}'@'%';

quit (to quit mysql cli)

exit (to exit bash)
```

## Create a database / schema
```
CREATE SCHEMA `{database name}` ;
```

## Time to test

Now that this has been set up, open up Mysql Workbench and connect to the database.

* Hostname = 127.0.0.1
* Port = 3306
* Username = username of the new user you created 

Then enter the password and you should now be connected and should see the database / schema you created.

## What now?

Well as a developer, you're always messing around with your database and its contents, and sometimes you mess things up. So it's handy to have a backup of your database ready to go. Thankfully this is really easy to do and restoring that database is just as easy.

## Backup
```
docker exec -i {container name} /usr/bin/mysqldump -u root -p {database or schema name} > {filenameToSaveAs}.sql
```
This backup script can now be found in what ever directory you are currently in.

## Restore
From the directory your backup script file is located.
```
docker exec -i {container name} /usr/bin/mysql -u root -p {database name} < {filenameToRestoreFrom}.sql
```

## Summary
So now I can spin up a Docker container with a Mysql server and database inside and have my backed up database ready to go. Nice and easy!