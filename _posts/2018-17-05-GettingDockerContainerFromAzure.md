---
title:  Get Docker container from Azure and run locally
author: Will Andrews
date: 2018-05-17
---

So I had a Docker image on an Azure container registry, and I wanted to pull it and run it locally on my machine.

So with command line I did the following:

Start Docker on my machine. (I always forget this....)

Login to the Azure registry.
```
sudo docker login <servername> -u <username> -p <password>
```

Pull the image that I want. (Tag is optional to get a specific version).

```
docker pull <servername>/<registry>/<image>:<tag>
```

Take a look at the images locally so that I can get the image id to start a container.

```
docker images
```

Then I grab the image id and run the run command, mapping the ports inside the container, to the ports on my machine, and giving it a name.
(80:80 means map port 80 inside the container, to port 80 on my machine.)

```
docker run -d -p 80:80 --name <give you container a name here> <image id>
```

Then I went to  http://localhost:80 and see my web app.

Few extra helpful commands.

View running containers:
```
docker ps
```

View all containers:
```
docker ps -a
```

Start container:
```
docker start <container id>
```

Stop container:
```
docker stop <container id>
```

Delete container:
```
docker rm <container id>
```

