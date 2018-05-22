---
title:  Pushing a local Docker image to an Azure container registry
author: Will Andrews
date: 2018-05-22
---

Let's say you've built a web app on your dev machine, created an Image for it, run it locally in a container and you're happy with how it's running. Now you need to deploy that to your container registry which happens to be on Azure. Not a problem.

First things first, you need to create a Docker image to use. To do that, use a 'Dockerfile' to build an image from your solution.

(From within the folder that your Dockerfile is located)
```
docker build .
```

Now run the following to find the image ID that has been created.

```
docker images
```

Make a note of the image ID as you will need that. 

Log into your Azure container.

```
sudo docker login <servername> -u <username> -p <password>
```

Now you need to tag the image you created.

```
docker tag <image id> <servername>/<repository folder>
```

Now when you run 'docker images' you will see that the image now has a repository and a tag.

Finally, the image needs to be pushed up to the Azure container registry so that it's available to use.

```
docker push <repository>
```

That's it. Log into your Azure container registry and take a look at the repositories you will see the new repository created and your image listed.
