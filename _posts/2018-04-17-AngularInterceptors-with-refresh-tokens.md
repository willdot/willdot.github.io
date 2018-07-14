---
title:  Angular Interceptors with some refresh token usage
author: Will Andrews
date: 2018-04-17
order: 4
---

Following on from the Refresh Token post, I thought I would do a quick how to on using this in an Angular app. I heard someone talking about interceptors in Angular and thought I would investigate as they sounded useful. Once I had figured out what they did and how simple they are, it made sense to use them to implement the refresh tokens.

Interceptors are a piece of code that get run on every http request that the app makes. This means, that if you are setting headers for authorization (like my app is), you can handle this code in one place for all http request. Nice. It then got me thinking, well if the interceptor receives a 401 result, then lets try and use the refresh token and then try the original http request again. 

So, first of all you need to create an interceptor.ts file with some code. I decided to add it into an _auth folder.

```typescript
// Basic setup
@Injectable()
export class TokenInterceptor implements HttpInterceptor {
  constructor(private router: Router, public auth: AccountService, private _http: HttpClient) {
  }
```
```typescript
    // Method that clones the incoming request and adds the headers to it.
    addHeaders(req: HttpRequest<any>, token: string): HttpRequest<any> {

    return req.clone({
        setHeaders: {
          Authorization: `Bearer ${token}`,
          'Content-Type': 'application/json'       
        }});
    }
```

This bit of code is doing quite a lot, but it send the request, adding the headers in. If it succeeds, great, if not, then if there is a 401 response, it tries to use the refresh token or if it's a 400 then it will log the user out.

```typescript
    /// When a http request is sent from a service, this method is called first and it 'intercepts' the request and does anything you tell it to.
  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {

    return next.handle(this.addHeaders(request, localStorage.getItem('id_token')))
    ///If there is an error response and it's 401, try and do th refresh token and then send the request again
    .catch((err, source) => {
        if (err instanceof HttpErrorResponse && err.status == 401) {
            return this.refreshToken()
            .concatMap(() => next.handle(this.addHeaders(request, localStorage.getItem('id_token'))).catch( error => {
                return Observable.throw('');
            }));
        }
        ///If it's a bad request, then log the user out as it's probably due to the refresh token not being valid.
        else if (err instanceof HttpErrorResponse && err.status == 400) {
            this.logOutUser();
            return Observable.throw('');
        }
    });
  }
```
```typescript
    // Send an api call to use a refesh token
  refreshToken() : Observable<any> {
      
    let tokenRequest :ITokenRefresh = {
        refreshToken : localStorage.getItem('refresh_token'),
        email: localStorage.getItem('currentUser')
    } 

    let body = JSON.stringify(tokenRequest);

    return this._http.post<ITokenRefresh>(this._authenicateUrl + 'UseRefreshToken', body)
        .do(
            data => {
                localStorage.setItem('id_token', data['token']);
                localStorage.setItem('token_expiry', data['expiry'].toString());
                localStorage.setItem('', data['refreshToken']);
                localStorage.setItem('refresh_token_expiry', data['refreshTokenExpiry'].toString());
            }
        )
        .catch( error => {
            return Observable.throw('');
        });
  }
```
```typescript
    // Log the user out
  logOutUser() {
    localStorage.setItem('currentUser', '');
    localStorage.setItem('id_token', '');
    localStorage.setItem('token_expiry', '');
    localStorage.setItem('refresh_token', '');
    localStorage.setItem('refresh_token_expiry', '');
    this.router.navigate(['/account']);
  }
```
```typescript
    // Handles an error response
    private handleError(err: HttpErrorResponse) {
        console.log(err.error);        
        return Observable.throw('');
    }
```

Also need to add this into the app.module files providers.

```typescript

  providers: [
    {
      provide: HTTP_INTERCEPTORS,
      useClass: TokenInterceptor,
      multi: true
    }
  ]

```

And that's it!

Example of this being used can be found in my DailyDo web app repo.
<https://github.com/willdot/DailyDo/tree/master/DailyDo.WebApplication>
