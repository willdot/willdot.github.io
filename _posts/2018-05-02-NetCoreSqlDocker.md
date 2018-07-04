---
title:  .Net Core + Local SQl + Docker + Mac
author: Will Andrews
date: 2018-05-02
order: 7
---

Started working through a PluralSight course on building .NET core API. Really good course and very thorough. 

It got to the stage where using a database was required, and the course used a local SQL database, something that isn't readily available on Visual Studio for Mac. So the work around that I found, which is great because I've been looking for a use for it, is to use Docker. FINALLY!! A use I've found for Docker!

So first thing I did was launch Docker, open up terminal and started to run some commands. Can't believe how easy all this was.

First, fetch the Docker container.
```
sudo docker pull microsoft/mssql-server-linux:2017-latest
```
Next run the container, accept the EULA and set the SA password (must be strong with lowercase, uppercase, number and special character).

```
sudo docker run -e 'ACCEPT_EULA=Y' -e 'MSSQL_SA_PASSWORD=PASSWORDGOESHERE' \
   -p 1433:1433 --name sql1 \
   -d microsoft/mssql-server-linux:2017-latest
```

Check that the container is running.
```
docker ps -a
```

That was it.... 

Then in my API code, I used the connection string such as
```
@"Data Source=localhost; Initial Catalog=DATABASENAME;User id=sa; password=PASSWORD;";
```

That was it. Super simple
