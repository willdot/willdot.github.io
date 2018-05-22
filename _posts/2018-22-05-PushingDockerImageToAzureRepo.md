---
title:  Pushing a local Docker image to an Azure container registry
author: Will Andrews
date: 2018-05-22
---

## Push and image from local to Azure

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
docker tag <image id> <servername>/<repository folder>:<optional tag>
```

Now when you run 'docker images' you will see that the image now has a repository and a tag.

Finally, the image needs to be pushed up to the Azure container registry so that it's available to use.

```
docker push <repository>:<optional tag name>
```

That's it. Log into your Azure container registry and take a look at the repositories you will see the new repository created and your image listed.

## Versioning

Let's say you wanted to have different versions. You've built something, want to push to to Azure and test it before you push it as the latest and overwrite the latest image.

No problem, just use tags.

Tag an image with 'V1'.

```
docker tag <image id> <servername>/<repository folder>:V1
```

Then push 

```
docker push <repository>:v1
```

Now when you look in your repository you will have your latest image in there from before, but you'll also have a new image tagged with V1.

Once you've checked that your test image worked, and you want to overwrite the latest, remove the optional tag (or use the tag 'latest') when tagging and pushing an image, and this will then overwrite the latest image tagged as latest in your repository.

This is handy for when you want to rollback. For example, you have 3 images in your repository:

* Latest (same as v2)
* V1
* V2

The latest is the same image as V2, but you realise there's a bug in it and you want to revert back to V1. You can pull V1 down locally, re-tag it as latest and then push it back up to overwrite the latest. Then you'll have the following 3 images in your repository, with one difference, the latest is the same as V1:

* Latest (same as v1)
* V1
* V2