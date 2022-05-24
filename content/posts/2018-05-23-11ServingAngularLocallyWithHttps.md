---
title:  Serving an Angular app locally over https
author: Will Andrews
date: 2018-05-17
order: 11
---

Had a bit of an issue trying to use

``` typescript
navigator.geolocation.getCurrentPosition
```
 with Safari as it wouldn't allow it over http So to use this, I need to serve my Angular app over https.

So first things first, I installed a package that contains a dev certificate.

```
npm install browser-sync --save-dev
```

Then in the package.json file, I replaced the start script to contain the following (or if you want to be able to use either http or https, then create a new start command such as 'start-https' and put the command in there.)

```
ng serve --ssl true -ssl-key /node_modules/browser-sync/lib/server/certs/server.key -ssl-cert /node_modules/browser-sync/lib/server/certs/server.crt
```
 Then running npm start, ran the above command, and there it was...my app running in https.