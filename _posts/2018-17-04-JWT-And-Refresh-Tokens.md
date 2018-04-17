---
title:  JWT and Refresh Tokens
author: Will Andrews
date: 2018-04-17
---

This is something I've been playing with for a while and I've had JWT implemented for a while in my web app, however something that always bugged me was 'How do you stop the user having to log in again, if the JWT expires after the short life span it's given?'. Refresh tokens. There seemed to be so many different ways to implement this, and it's a sea of confusion so for my needs, I created a hybrid of some of them. 

Theory:

1: User logs in.

2: Api returns a JWT and a refresh token (Which is a GUID). The API saves the refresh token in a database, with the users email, an expiry and if the token it active (more on that in a bit).

3: The web app stores the tokens in the local storage and each time an API call is made, sends the JWT as the header, which the API will authenticate.

4: If the token has expired, then a 401 status is sent back to the web app, so it knows to try the refresh token.

5: The web app sends the refresh token to an api method, and as long the token is active in the database, not past it's expiry date and for the correct user, then a new JWT and refresh token are sent back to the web app to use and retry the failed api call again.

6: The api deactivates the old refresh token, so that it can't be used again, but here's the cool bit. If at any point, I fear that there is a breach and the refresh tokens that are currently active are in danger, I can set all the refresh tokens in the database to be inactive, and every user will have to log in again.


Take a look at my DailyDo git repo for how I implemented this. The AuthenticationTokenController is where most of the magic happens.


https://github.com/willdot/DailyDo/tree/master/DailyDo.WebApi