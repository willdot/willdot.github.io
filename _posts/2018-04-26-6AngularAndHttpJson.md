---
title:  Angular - Http JSON responses
author: Will Andrews
date: 2018-04-26
order: 6
---

Interesting morning. I was retrieving some data from an api in Angular and then trying to use .Filter to filter an array of returned objects. My class that I had mapped the JSON response to had property names using Pascal case, my API objects in the .Net Core project also had the property names using Pascal case. So when I was trying to do 
```typescript
.filter(a => a.Week == weekNumber && a.SeasonType == seasonType);
```
I was getting no objects returned from my filter. After a bit of head banging against my desk, I noticed that the JSON data was using camel case for the property names.

So I made a change and tried this instead, just to see what the outcome was.
```typescript
.filter(a => a['week'] == weekNumber && a['seasonType'] == seasonType);
```

It worked, interesting. So instead of doing that, I went through all my Angular classes an changed all the property names to use camel case, switched back to my original filter code and everything worked.


Apparently, .Net api defaults to returning JSON in camel case, but there is a way to change this! Will give this a go later and see what happens.