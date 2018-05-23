---
title:  Serving an Angular app locally over https
author: Will Andrews
date: 2018-05-17
---

Had a bit of an issue trying to use 'navigator.geolocation.getCurrentPosition' with Safari as it wouldn't allow it over http So to use this, I need to serve my Angular app over https.

So first things first, I installed a package that contains a dev certificate.

```
npm install browser-sync --save-dev
```

Then in the package.json file, I replaced the start script to contain the following.

```
ng serve --ssl true -ssl-key /node_modules/browser-sync/lib/server/certs/server.key -ssl-cert /node_modules/browser-sync/lib/server/certs/server.crt
```
 Then running npm start, ran the above command, and there it was...my app running in https.