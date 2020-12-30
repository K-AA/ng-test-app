Introduction
Hello! Today we will talk about server side rendering (SSR) tuning for Angular.

In this article you will learn:

angular SSR setup
HttpClient rehydration
auth during SSR
angular "native" i18n support setup
Let's go!
I assume that your already have @angular/cli installed.

We will start from scratch. First create new project:
ng new playground
cd playground
Then run the following CLI command
ng add @nguniversal/express-engine
Now, we have a couple of new files:
main.server.ts - bootstrapper for server app
app.server.module.ts - server-side application module
tsconfig.server.json - typescript server configuration
server.ts - web server with express

Let's refactor our server.ts file a little bit:
import "zone.js/dist/zone-node";

import { ngExpressEngine } from "@nguniversal/express-engine";
import * as express from "express";
import * as path from "path";

import { AppServerModule } from "./src/main.server";
import { APP_BASE_HREF } from "@angular/common";
import { existsSync } from "fs";

const server = express(); // express web server
const baseHref = "/"; // will be needed in future, to handle different bundles for i18n

// folder where angular put browser bundle
const distFolder = path.join(process.cwd(), "dist/playground/browser"); 

// ref for index.html file
const indexHtml = existsSync(path.join(distFolder, "index.original.html")) ? "index.original.html" : "index";

// just port for our app :)
const port = process.env.PORT || 4000;

// This is the place where all magic things happens. 
// Actually, it is middleware which use universal CommonEngine
// for building html template for request
server.engine("html", ngExpressEngine({ bootstrap: AppServerModule }));
server.set("view engine", "html");
server.set("views", distFolder);

// helps to serve static files from /browser
server.use(baseHref, express.static(distFolder, { maxAge: "1y", index: false }));

server.get("*", (req, res) => {
  const requestInfo = new Date().toISOString() + ` GET: ${req.originalUrl}`;
  console.time(requestInfo);

  res.render(indexHtml,
    { req, providers: [{ provide: APP_BASE_HREF, useValue: baseHref }] },
    (error, html) => {
      if (error) console.log(error);
      res.send(html);
      console.timeEnd(requestInfo);
    });
});
server.listen(port, () => {
  console.log(`Node Express server listening on http://localhost:${port}`);
});

export * from "./src/main.server";
And, that's all! Now we can build and run our project. But...
To say the truth, not everything is so simple as it seems to be.
And I will show you why.

HttpClient rehydration
Create core.module.ts with custom-http-client.service.ts in it.

custom-http-client.service.ts
import { Injectable } from "@angular/core";
import { HttpParams, HttpClient } from "@angular/common/http";
import { Observable } from "rxjs";

@Injectable()
export class CustomHttpClientService {

  constructor(private httpClient: HttpClient) { }

  get<T>(path: string, params?: HttpParams): Observable<T> {
    return this.httpClient.get<T>(path, 
      { observe: "body", responseType: "json", params: params });
  }
}
core.module.ts
import { NgModule } from "@angular/core";
import { HttpClientModule } from "@angular/common/http";
import { CustomHttpClientService } from "src/app/core/custom-http-client.service";

@NgModule({
  imports: [HttpClientModule],
  providers: [CustomHttpClientService]
})
export class CoreModule {}
Then, import core.module.ts to app.module.ts.
And also, modify app.component.ts
import { Component, OnInit } from '@angular/core';
import { CustomHttpClientService } from "src/app/core/custom-http-client.service";

interface User {
  name: string;
  email: string;
  website: string;
}

@Component({
  selector: 'app-root',
  template: `
    <div>
      <h1>Users List</h1>
      <div *ngIf="users && users.length">
        <div *ngFor="let user of users">
          <div>Name: {{user.name}}</div>
          <div>Email: {{user.email}}</div>
          <div>Site: {{user.website}}</div>
        </div>
      </div>
    </div>
  `,
  styleUrls: ['./app.component.css']
})
export class AppComponent implements OnInit {

  users: User[];

  constructor(private http: CustomHttpClientService) { }

  ngOnInit(): void {
    this.http.get<User[]>("https://jsonplaceholder.typicode.com/users")
      .subscribe(users => {
        this.users = users;
      });
  }
}
Run the following command
npm run build:ssr
npm run serve:ssr
Then, open your browser at http://localhost:4000
And now, you can see strange things happens.
First browser receive html from web server and after renders html one more time at clientside. It is default behavior for angular. Because client side angular does not know anything about server side rendering. To solve this issue, Angular Universal provides TransferState store. When this store is in use, the server will embed the data with the initial HTML sent to the client.

Let's modify our codebase.

custom-http-client.service.ts
import { Injectable, Inject, PLATFORM_ID } from "@angular/core";
import { HttpParams, HttpClient } from "@angular/common/http";
import { Observable, of } from "rxjs";
import { tap } from "rxjs/operators";
import { StateKey, makeStateKey, TransferState } from "@angular/platform-browser";
import { isPlatformServer } from "@angular/common";

@Injectable()
export class CustomHttpClientService {

  constructor(
    private httpClient: HttpClient,
    private transferState: TransferState,
    @Inject(PLATFORM_ID) private platformId: Object,
  ) { }

  get<T>(path: string, params?: HttpParams): Observable<T> {
    const transferKey: StateKey<T> = makeStateKey(`${path}?${params != null ? params.toString() : ""}`);

    if (this.transferState.hasKey(transferKey)) {
      return of(this.transferState.get<any>(transferKey, 0))
        .pipe(
          tap(() => this.transferState.remove(transferKey))
        );
    } else {
      return this.httpClient.get<T>(path, { observe: "body", responseType: "json", params: params })
        .pipe(
          tap(response => {
            if (isPlatformServer(this.platformId)) {
              this.transferState.set<T>(transferKey, response);
            }
          })
        );
    }
  }
}
app.module.ts
...

@NgModule({
  imports: [
    BrowserModule.withServerTransition({ appId: 'serverApp' }),
    BrowserTransferStateModule,
    CoreModule,
  ],
  declarations: [
    AppComponent
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
app.server.module.ts
...

@NgModule({
  imports: [
    AppModule,
    ServerModule,
    ServerTransferStateModule,
  ],
  bootstrap: [AppComponent],
})
export class AppServerModule {}
Now, if we build and run our app, we will see that angulr does not do double work and html received from the web server does not rendered for the second time.

But how actually this works? During server side rendering angular includes the data from the TransferState store to the script tag in the html string which sends to the client. You can verify this by simply looking in network tab.

Auth during SSR
There are two common ways of dealing with user authentication - json web token based and session-based authentication.

In this article I want to show how to handle the second approach, with sessions.

First of all, let's add a cookie-parser middleware to our web server. It will parse incoming request and attach cookie string to request object.
npm i --save cookie-parser
server.ts
... 
import * as cookieParser from "cookie-parser";

...
server.engine("html", ngExpressEngine({ bootstrap: AppServerModule }));
server.set("view engine", "html");
server.set("views", distFolder);
server.use(cookieParser());
Then, modify our app.server.module to get access to request object from the express web server.

app.server.module
...
import { REQUEST } from "@nguniversal/express-engine/tokens";
import { Request } from "express";

@Injectable()
export class IncomingServerRequest {
  constructor(@Inject(REQUEST) private request: Request) { }

  getCookies() {
    return !!this.request.headers.cookie ? this.request.headers.cookie : null;
  }
}

@NgModule({
  imports: [
    AppModule,
    ServerModule,
    ServerTransferStateModule,
  ],
  bootstrap: [AppComponent],
  providers: [
    { provide: "INCOMING_REQUEST", useClass: IncomingServerRequest },
  ]
})
export class AppServerModule {}
Then, create cookies.interceptor.ts

cookies.interceptor.ts
import { HttpInterceptor, HttpRequest, HttpHandler, HttpEvent } from "@angular/common/http";
import { Optional, Inject, PLATFORM_ID, Injectable } from "@angular/core";
import { Observable } from "rxjs";
import { isPlatformServer, isPlatformBrowser } from "@angular/common";

@Injectable()
export class CookiesInterceptor implements HttpInterceptor {

  constructor(
    @Inject(PLATFORM_ID) private platformId: Object,
    @Optional() @Inject("INCOMING_REQUEST") private request: any
  ) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    if (isPlatformServer(this.platformId) && this.request) {
      const requestCookies = this.request.getCookies();

      if (requestCookies) {
        req = req.clone({setHeaders: {Cookie: requestCookies}});
      }
    }

    if (isPlatformBrowser(this.platformId)) {
      req = req.clone({ withCredentials: true })
    }

    return next.handle(req);
  }
}
and provide it in core.module.ts

core.module.ts
import { NgModule } from "@angular/core";
import { HttpClientModule, HTTP_INTERCEPTORS } from "@angular/common/http";
import { CustomHttpClientService } from "src/app/core/custom-http-client.service";
import { CookiesInterceptor } from "src/app/core/cookies.interceptor";

@NgModule({
  imports: [HttpClientModule],
  providers: [
    CustomHttpClientService,
    {
      provide: HTTP_INTERCEPTORS,
      useClass: CookiesInterceptor,
      multi: true,
    }
  ]
})
export class CoreModule {}
Now, if we build and run our app, we will see a message Refused to set unsafe header "Cookie". That happens, because XMLHttpRequest does not allow to set cookie headers manually. Fortunately, we can dodge this adding some code to server.ts

Note: Actually, this monkey patching breaks XMLHttpRequest Content Security Policy. So this code MUST be only in server bundle. Do not use this hack in browser.
server.ts
...
import * as xhr2 from "xhr2";

xhr2.prototype._restrictedHeaders = {};

const server = express(); // express web server
...
Now, if you build and run your app, the behavior will be as it should.

i18n support setup
First, install some packages for localization.
npm i --save @angular/localize
npm i --save-dev ngx-i18nsupport
Then, add xliffmerge.json file to the root folder.

xliffmerge.json
{
  "xliffmergeOptions": {
    "srcDir": "src/i18n",
    "genDir": "src/i18n",
    "i18nFile": "messages.xlf",
    "i18nBaseFile": "messages",
    "i18nFormat": "xlf",
    "encoding": "UTF-8",
    "defaultLanguage": "en",
    "languages": [
      "ru"
    ],
    "removeUnusedIds": true,
    "supportNgxTranslate": false,
    "ngxTranslateExtractionPattern": "@@|ngx-translate",
    "useSourceAsTarget": true,
    "targetPraefix": "",
    "targetSuffix": "",
    "beautifyOutput": false,
    "allowIdChange": false,
    "autotranslate": false,
    "apikey": "",
    "apikeyfile": "",
    "verbose": false,
    "quiet": false
  }
}
Modify angular.json, to handle english locale as default and russian as additional. I highly recommend to copy-paste from this source because the actual size of file is too big for this article.

And also modify app.component.ts's html template

app.component.ts
template: `
    <div>
      <h1 i18n="@@usersListTitle">Users List</h1>
      <button i18n="@@getUsersButton">Get Users</button>
      <div *ngIf="users && users.length">
        <div *ngFor="let user of users">
          <div>Name: {{user.name}}</div>
          <div>Email: {{user.email}}</div>
          <div>Site: {{user.website}}</div>
        </div>
      </div>
    </div>
  `,
with directive i18n we can mark places where translation will be used

Then, add new command to "scripts" in package.json file and execute.

package.json
"extract-i18n": "ng xi18n --output-path src/i18n --out-file messages.xlf && xliffmerge --profile ./xliffmerge.json"
If you did everything right, you will receive a message:
WARNING: please translate file "src/i18n/messages.ru.xlf" to target-language="ru"
Now, we have two language locales and two different builds, but one server.ts file. We need to refactor it a little bit, to handle this situation.

server.ts
...
const server = express();
const language = path.basename(__dirname); // take folder name "en" or "ru" as current language
const baseHref = language === "en" ? "/" : `/${language}`;
const distFolder = path.join(process.cwd(), "dist/browser", language);
...
and then add two new commands to "scripts" in package.json file
...
"serve:ssr:en": "node dist/server/en/main.js",
"serve:ssr:ru": "node dist/server/ru/main.js",
...
Now we have one build command for all locales and our starter is ready to go!
