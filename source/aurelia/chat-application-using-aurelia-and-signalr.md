# Chat Application using Aurelia and SignalR

_Published by [shahabganji](https://shahabganji.me) on October 8, 2018_

Hello and welcome! You have definitely heard of SignalR for real-time web applications, in this post I want to help you write a simple traditional chat application using SignalR and my favorite SPA framework [Aurelia](https://aurelia.io). You can download the source code from my GitHub, [here](https://github.com/shahabisblogging/ChatApplicationAureliaSignalR), notice that in this sample we use WebPack for bundling.

We need to create a new solution in which we have both ASP.NET Core and Aurelia configured; thus I use `AureliaToolbelt.AspNetCore::1.2.1` spa template, it was created by Aurelia CLI, you must install it beforehand: 

```bash
dotnet new --install AureliaToolbelt.AspNetCore::1.2.1
```

or use your own preferred choice, for instance, Aurelia CLI, then, create a directory named **SignalRChatApp** and run the following command:

```bash
dotnet new aut
```

it takes some time to have all dependencies installed, let’s wait :smile:

Well, this application is two fold: Backend using asp.net core and SignalR and frontend using Aurelia. Let’s begin with the former.

## Backend, SignalR Hubs

We need to bootstrap SignalR so that we can create hubs and connect to them via a valid URL that we have defined. Hubs are the endpoints in SignalR abstracting the underlying connection layer for real-time communication; that could be WebSockets, or long polling or etc.

First, we need to tell the DI system which services we require, to do that add the following code in `ConfigureServices` method in the `Startup' class:

```csharp
services.AddSignalR( );
```

then, add SignalR middleware to the pipeline, bear in mind to add SignalR’s middleware before the MVC middleware, if not, make sure that your SignalR’s routes will not be caught by MVC routes.

```csharp
 app.UseSignalR( routes => {
    routes.MapHub<ChatHub>( "/chat" );
 });
```

In the previous code snippet, we register a route for our chat room endpoint, `/chat`, map it to a hub named ChatHub, it’s time for our hub.

Create a folder in the root directory and name it **Hubs**, then create a C# class, `ChatHub.cs`, copy the following snippet:

```csharp
public class ChatHub : Hub {
    public Task Send(string sender, string message) {
      return this.Clients.All.SendAsync("updateMessages", new { sender, message });
    }
}
```

We created a hub with a simple method taking two arguments, sender and the message, which is called from client application later on, and will call the `updateMessage` on all clients connecting to this endpoint. Now, it’s time to take a look at the other side of the argument, client, Aurelia.

## Aurelia, Front-end player

The main player here is a service which is responsible for the communication between our Aurelia app and ChatHub, `chat-hub-service.ts`, BTW I am a fan of typescript so that’ll be used. And since I never like an application without authorization, I implemented a fake authentication in this demo and its implemented by using [AuthorizeStep](https://aurelia.io/docs/routing/configuration#pipelines) in conjunction with another service, `auth-service.ts`;  the latter is a class with methods and properties to ease the login process, for now, it just checks localStorage for the logged-in user, you can then replace it with a more complex service which checks for instance through an identity server, it has two methods, `login` and `logout`,and two properties.


```ts
import { singleton } from "aurelia-framework";
 
@singleton()
export class AuthService {
 
  private loggedInKey: string = "logged-in";
 
  public get isLoggedIn(): boolean {
    return localStorage.getItem(this.loggedInKey) ? true : false;
  }
 
  public get username() {
    return localStorage.getItem(this.loggedInKey);
  }
 
  public login(username: string): Promise<boolean> {
 
    return new Promise(resolve => {
 
      localStorage.setItem(this.loggedInKey, username);
      resolve(true);
    });
 
  }
 
  public logout(): Promise<boolean> {
    return new Promise(resolve => {
      localStorage.removeItem(this.loggedInKey);
      resolve(true);
    });
  }
}
```

We require this service in multiple places, first of which is our Authorization step within Aurelia’s pipeline, change your **app.ts** file to the following, I have **App** class and **AuthorizeStep** class in the same file, that would be better if you have them in separate files.


```ts
import "toastr/build/toastr.css";
import "font-awesome/css/font-awesome.css";
 
import { ToastrService } from 'aurelia-toolbelt';
	
import { PLATFORM } from "aurelia-pal";
import { autoinject } from 'aurelia-dependency-injection';
import { RouterConfiguration, Router, Redirect } from 'aurelia-router';
 
import { AuthService } from './services/auth-service';
 
@autoinject()
export class App {
 
  private router: Router;
  public message: 'Hello World!';
 
  constructor(private ts: ToastrService) { }
 
  private configureRouter(config: RouterConfiguration, router: Router): void {
 
    this.router = router;
 
    config.title = 'Chat';
    config.addAuthorizeStep(AuthorizeStep);
 
    config.map([
      { route: ['login', ''], name: 'login', moduleId: PLATFORM.moduleName('./login/login'), nav: true, title: 'Login' }
      ,
      { route: 'chat-room', name: 'chat-room', moduleId: PLATFORM.moduleName('./chat/chat-room'), nav: true, title: 'Chat Room', settings: { auth: true } }
    ]);
  }
}
 
@autoinject()
class AuthorizeStep {
 
  constructor(private authService: AuthService) { }
 
  run(navigationInstruction, next) {
 
    if (navigationInstruction.getAllInstructions().some(i => i.config.settings.auth)) {
      if (!this.authService.isLoggedIn) {
        return next.cancel(new Redirect('login'));
      }
    }
 
    return next();
  }
}

```

As you can see in the above code snippet, we have two routes configured, **login**, and **chat-room** which requires authentication, that means when we add an authorize step in the Aurelia pipeline and set the **auth** property of a rout to **true**, then that step is responsible for validating if the user is currently authenticated or not, in this case the authorization step uses **AuthService**'s **isLoggedIn** property for validation; if fails, it will redirect the user to login page.

The next step is to declare a class which is responsible to communicate with the hub, that’s [chat-hub-service.ts](https://github.com/shahabisblogging/ChatApplicationAureliaSignalR/blob/master/src/services/hubs/chat-hub-service.ts)

First of all we need to add [@aspnet/signalr](https://www.npmjs.com/package/@aspnet/signalr) npm package to do so do one of the following:

```bash
npm i @aspnet/signalr 
-------- OR ---------
yarn add @aspnet/signalr
```

then you need to create a connection via `HubConnectionBuilder`.

```ts
import { HubConnectionBuilder, LogLevel, HubConnection } from '@aspnet/signalr';
	
@singleton()
@autoinject()
export class ChatHubService {
 
  private connection: HubConnection;
	
constructor() {
    this.connection = new HubConnectionBuilder()
      .withUrl('/chat')
      .configureLogging(LogLevel.Information)
      .build();
 
    this.connection.on("updateMessages", (data) => this.notifier(data));
  }
}
```

I have to mention that this service is a singleton so that only one instance will be available in the whole application, but what if we need multiple places to be notified when a message received? That lies in the heart of using `EventAggregator` and that is in the **notifier** method, you need to update constructor to inject/autoinject an instance of  `EventAggregator`

```ts
constructor(private authService: AuthService, private ea: EventAggregator) {
	...
}
	
private notifier(data: any) {
    console.log(data);
    this.ea.publish("Message-Received", data);
}
```

Now wherever we need to get messages from the server side hub we can use `EventAggregator` to subscribe to `Message-Received` event, and we’ll be notified. Thanks to Aurelia messaging system :smile: .

There’s still on thing left, we have created our connection, we have not connected to the hub though. you need to call start method on your connection, hence I create a **start** method within the `ChatHubService` in order for users of this service to be able to start the connection on demand. The same goes for **stopping** the connection, in the above-mentioned methods we return promises, that enables the clients of the service to register callbacks to handle failure and success.

```ts
public start() {
    return this.connection.start().catch(err => console.error(err.toString()));
}
	
public stop() {
    return this.connection.stop();
}
```

The last piece of code for this service enables us to send messages from client to server, in other words calling server methods on our hub from our Aurelia application. You should call the invoke method on your connection, the first parameter is the name of the method on your C# hub, and the rest are the input parameters for that method, they should match the server side method arguments respectively.


```ts
public sendMessage(message: string) {
    return this.connection.invoke("Send", this.authService.username, message);
}
```

We use **AuthService** here to send the logged-in username as the sender of the message to the server, and the next parameter is the message.

In the [chat-room](https://github.com/shahabisblogging/ChatApplicationAureliaSignalR/tree/master/src/chat) module we will use the `chat-hub-service` to send and get messages to and from other users.

```ts
import { autoinject } from "aurelia-framework";
import { AuthService } from "services/auth-service";
import { Router } from "aurelia-router";
 
import { ChatHubService } from "services/hubs/chat-hub-service";
import { EventAggregator } from "aurelia-event-aggregator";
 
@autoinject()
export class ChatRoom {
 
  private message: string = null;
  private messages = [];
 
  private hasFocus = true;
 
  constructor(private authService: AuthService, private router: Router, private chatService: ChatHubService, private ea: EventAggregator) { }
 
  private canActivate() {
    // if the start fails we won't land in this page
    return new Promise((resolve, reject) => {
      this.chatService.start().then(_ => {
        this.ea.subscribe("Message-Received", (data) => {
          this.messages.push(data);
        });
        resolve(true);
      }).catch(_ => {
        resolve(false);
      });
    });
  }
 
  private logout() {
    this.authService.logout().then(response => {
      this.router.navigateToRoute("login");
    });
  }
 
  private addMessage() {
    return new Promise(resolve => {
      this.chatService.sendMessage(this.message).then(res => {
        this.message = "";
        this.hasFocus = true;
        resolve(true);
      });
    });
  }
 
  detached() {
    // when we move out of this page we want it to stop
    this.chatService.stop();
  }
 
}
```

Instances of `AuthService`, `ChatHubService` and `Router` are injected. Inside the `canActivate` method we check whether it is possible to start the connection, if not, we won’t route to this page, a better approach could be displaying an alert message; otherwise, we subscribe to `Message-Received` event in order to be notified when a message pushed from the server via this connection. On `addMessage` click_handler we use injected `chat-hub-service` to publish messages to the server which will then push them back to all the connected clients.

On our `detached` method we just stop our connection, one may not need to do so, to show notification toasts for instance on receiving messages.

The `html` part consists of a `text box` binding to the message property, and a button's click bound to `addMessge`. <li> elements create for each message received from the server.


```html
<template>
   <abt-alert type="info">
    <div>
      <abt-button bs-type="danger" click.call="logout()">
        Logout
      </abt-button>
    </div>
  </abt-alert>
 
  <div>
    <form>
      <abt-inputgroup class="mb-3">
        <input type="text" class="form-control" placeholder="type your message here" value.bind="message" focus.bind="hasFocus">
        <abt-inputgroup-append>
          <abt-button type="submit" bs-type="secondary" click.call="addMessage()">
            Send
          </abt-button>
        </abt-inputgroup-append>
      </abt-inputgroup>
    </form>
    <br>
 
    <ul>
      <li repeat.for="message of messages">
        ${message.sender}:     ${message.message}
      </li>
    </ul>
  </div>
 </template>
 
```

I hope you find this article useful from both viewpoints, Aurelia and SignalR.



