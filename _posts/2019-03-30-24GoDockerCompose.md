---
title:  Hosting a web application and MySql database in a Docker container with Docker Compose
author: Will Andrews
date: 2019-03-30
order: 24
published: true
---

So far when ever I create an application and then use Docker, I run a Docker command that uses a DockerFile to create and image and then I have to spin up a new container for that image. But what I wanted was to be able to run a command and for my existing container to get the latest version of the code I was developing. That's where I read about Docker Compose.


## Background info on my app

I created a Go http server app. Nice simple code. Heading to http://localhost:3001/home will display "Hello"

``` go
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"os"

	"github.com/gorilla/mux"
)

func main() {
	var PORT string
	if PORT = os.Getenv("PORT"); PORT == "" {
		PORT = "3001"
	}
	router := mux.NewRouter()

	router.HandleFunc("/home", home).Methods("GET")

	err := http.ListenAndServe(":"+PORT, router)

	if err != nil {
		log.Fatal(err)
	}

}

func home(w http.ResponseWriter, r *http.Request) {

	hello := "Hello"
	json.NewEncoder(w).Encode(hello)
}
```

I've also used "https://github.com/kardianos/govendor" to create my import vendors folder, so that there's no need for go get in docker

## It starts with a Dockerfile

The first thing I do is create my Dockerfile which pulls the latest golang image, copies my code and sets an ENV variable for a port. Nice simple stuff.

``` dockerfile
FROM golang:1.12

ADD . /go/src/app
WORKDIR /go/src/app

ENV PORT=3001

```

So this pulls the latest Golang image, adds everything from the current directory into the "go/src/app" folder, sets that folder as a working directory. Then sets an environment variable to set the port. (Remember in the Go code, there was logic in the main func to get this or use 3001 by default).

## Enter Docker compose

Docker Compose is slightly different. It allows lots of builds to be done at once, with once command and one configuration. For example, my web app needs a database, and I'm using a MySql database Docker image, I can configure the docker-compose file to create the web app and then the Mysql image, and even configure the connection between the 2. I'm just going to create my web app first.

``` yml
version: '2'
services:
  app:
    container_name: my-app
    build: .
    command: go run main.go
    volumes:
      - .:/go/src/app
    working_dir: /go/src/app
    ports:
      - "60:3001"
    environment:
      PORT: 3001
```

This is similar to a dockerfile. Under services, you can define and app. Under app you give your container a name, tell it to build, what command you want to run when the container starts up.

You then give it a volume. 
``` yml
.:/go/src/app
```
The above will copy everything from the current directory (sub folders included) and put them into the "go/src/app" folder in the container.

Next you specify the working directory of the container, which is the folder we copied all the files into. Then you specify any port mappings. I'm mapping port 60 on my local machine to port 3001 on the container.

## Spin it up!

Now that the config is set up, we can run the commands to get things running.

First run the command to build
```
docker-compose up --build
```

You will see that you app has been created and if you run docker ps -a in another terminal you will see it open. Then head to localhost:60 and see you app running.


## Adding MySql into the mix

Now we have our web app up and running, we can add in a database. My web app doesn't use a database yet, but let's pretend it does.

Configuration looks like this:

``` yml
version: '2'
services:
  app:
    container_name: my-app
    build: .
    command: go run main.go
    volumes:
      - .:/go/src/app
    working_dir: /go/src/app
    ports:
      - "60:3001"
    environment:
      PORT: 3001
  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: 'db'
      MYSQL_USER: 'user'
      MYSQL_PASSWORD: 'password'
      MYSQL_ROOT_PASSWORD: 'password'
    ports:
      - '3306:3306'
    expose:
      # This will open port 3306 on the container
      - '3306'
      # Where our data will be persisted
    volumes:
      - my-db:/var/lib/mysql
# Names our volume
volumes:
  my-db:
```

As you can see it's not that complicated. Tell it to pull an image, set some database enviornment variables for username and passwords. Then set the ports to be used, then expose the container port. Finally add the volume which is where the data will be stored.
 
```
docker-compose up --build -d
```
Running this command will not rebuild your web app, because nothing changed, but then create a container for the MySql database. 

If you make some changes to the web app code and re run that command you'll notice that it will only recreated the web app container, but leave the MySql one alone. You can test this by opening up that Database with MySql Workbench, creating a table. Then change your web app code and run the build command. You can also run:
```
docker-compose down
```

This will stop all containers. Then when you run the up command you will notice it will start the container and your database instance will persist and still have the table you created.

Using this, along with the backup MySql docker database post I did previously will allow you to keep your test data.

## Helpful commands

```
// Start the container if it's already been built and pick up from where you left off
docker-compose up
```

```
// Rebuild the container (this will remove everything previously in the container)
docker-compose up --build
```

```
// using the -d flag will tell it to run disconnected 
docker-compose up -d

or

docker-compose up --build -d
```

```
// This will show you any previous images that are no longer in use
docker images -f 'dangling=true'

// This will then remove all these images
docker rmi $(docker images -aq -f 'dangling=true')
```