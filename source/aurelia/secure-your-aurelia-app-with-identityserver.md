# Secure your Aurelia app with IdentityServer

_Published by [shahabganji](https://shahabganji.me) on October 22, 2018_

Hello and welcome! Today I want to describe how we can secure our [Aurelia](https://aurelia.io) applications with the help of an Authorization Server, in this case, [IdentityServer4](https://identityserver.io/). I hope you enjoy 🙂 the topic and find it fruitful for your applications. The topic is twofold, how to run IdentityServer4, and how to configure Aurelia to be able to communicate with our ISP( Identity Service Provider ). We are using one of the awesome plugins of Aurelia to do so; [aurelia-open-id-connect](https://github.com/aurelia-contrib/aurelia-open-id-connect), many thanks to Mr. [Shaun Luttin](https://github.com/shaunluttin) for all his efforts. You can download the source code of this post from my [Github](https://github.com/shahabisblogging/AureliaIdentityServer) repository.

## Identity Server

The identity server is an Identity Service Provider written for .Net developers. It provides solutions such as single sign-on, identity management, API security and so on for your modern applications. As they say: “The Identity and Access Control solution that works for you”. Let’s begin. Firstly, you need to create an empty Asp.Net Core application by running,

```bash
dotnet new web --name Aurelia.IdentityServer
```

Then you need to add NuGet packages to your project.

```bash
dotnet add package IdentityServer4 --version 2.2.0
```

That being installed, you are ready to configure your own ISP using Identity Server. We need to declare `TestUsers`, `Clients`, `IdentityResources`, and `ApiResource`. To do so, define a class named `IdentityConfiguration` and copy the following code:

```cs
public static class IdentityServerConfiguration
{
	public static IEnumerable<IdentityResource> IdentityResources =>
		new List<IdentityResource> {
			new IdentityResources.OpenId() ,
			new IdentityResources.Profile()
		};
	
	public static IEnumerable<ApiResource> ApiResources =>
		new[] {
			new ApiResource( "aurelia_web_api" , "Aurelia WebApi") {
				ApiSecrets = { new Secret( "apisecret".Sha256() ) }
				}
		};

	public static List<TestUser> Users => new List<TestUser>() {
		new TestUser() {																	 
			SubjectId = "1D9F016D-58A9-4256-85A1-188ACE29DB44",
       			Username = "shahab" ,
		       Password = "password"													}
		};
	
	// those who want to get access to protected resources, such as api or identity resources
	public static IEnumerable<Client> Clients => new List<Client>(){
		new Client() {
			ClientId = "aurelia_web_api_client_spa",
			ClientName = "Aurelia SPA Application",

			AllowedGrantTypes = GrantTypes.Implicit,
			AllowAccessTokensViaBrowser = true ,

			RedirectUris = { "https://localhost:44347/signin-oidc" } ,
			PostLogoutRedirectUris = { "https://localhost:44347/signout-oidc" },
			AllowedCorsOrigins = { "https://localhost:44347" } ,

			AllowedScopes = new List<string>() {
				IdentityServerConstants.StandardScopes.OpenId,
				IdentityServerConstants.StandardScopes.Profile ,
				"aurelia_web_api"
				}
			}
		};
	}
```

There are several things you need to know in the above code snippet, first, we declared the resources we want to protect against unauthorized users, that includes both `Identity Resources` and `Api Resources`. [These lines](https://github.com/shahabisblogging/AureliaIdentityServer/blob/master/Aurelia.IdentityServer/IdentityServerConfiguration.cs#L10-L21) define those resources. Next, we should define users, in this demo, they are `TestUsers`, and then the most important part is to specify which clients are requesting for authentication. Before getting into details of the clients it is worthy to note that the user has a `SubjectId` which must be unique in this server. Clients have so many important properties, let’s check them one by one:

* **ClientId:** This is the client’s identifier which must be unique.
* **ClientName:** This is a human-readable name for the client.
* **[AllowedGrantTypes](http://docs.identityserver.io/en/latest/topics/grant_types.html):** Indicates that how this client can interact with the Identity Server. There are several grant types defined in OpenID Connect and OAuth2 specs, of which we use Implicit that is optimized for browser-based applications.
* **[AllowAccessTokensViaBrowser](http://docs.identityserver.io/en/latest/reference/client.html#basics):** Specifies whether this client is allowed to receive access tokens via the browser.
* **RedirectUris:** Where you want the identity server to return the tokens after a successful login.
* **PostLogoutRedirectUris:** Allowed URIs after logout to return to.
* **AllowedCorsOrigins:** If your client is on a different domain than your ISP, then you must specify this property to prevent Cross-Origin Attacks.
* **AllowedScopes:** List of resources, identity and API, that this client is allowed to request. The user can also at the consent page restrict them by checking or unchecking these resources.

After setting those properties properly, we need to configure the IdentityServer4 in our applications, first of all, you need to add the following code snippet to your `ConfigureServices` method of your `Startup` class.

We have added a developer signing certificate, and then we have added TestUsers, Clients and Resources defined earlier on. Next step is to add the **Authentication** middleware to the pipeline, go to your `Configure` method and paste `app.UseIdentityServer()` just before the mvc middleware.

```cs
public class Startup
{
	public void ConfigureServices(IServiceCollection services)
	{
		services.AddMvc();

		services.AddIdentityServer()
			.AddDeveloperSigningCredential()
			.AddTestUsers(IdentityServerConfiguration.Users)
			.AddInMemoryClients(IdentityServerConfiguration.Clients)
			.AddInMemoryApiResources(IdentityServerConfiguration.ApiResources)
			.AddInMemoryIdentityResources(IdentityServerConfiguration.IdentityResources);
	}

	public void Configure(IApplicationBuilder app, IHostingEnvironment env)
	{
    
		app.UseIdentityServer();
		app.UseStaticFiles();
		app.UseMvcWithDefaultRoute();
	}
}
```

So far so good. One thing is left, and that is to add basic user interfaces for your Identity Server, to do so use the guidance from their [official GitHub repository](https://github.com/IdentityServer/IdentityServer4.Quickstart.UI#adding-the-quickstart-ui). Now, in order to run the Identity Server execute `dotnet run --project Aurelia.IdentityServer` or hit **F5**. You must be able to see the Home page of the project click on Discovery Link to see more details.

## Aurelia Side

For this part you have two options, using `oidc-client-js` or `aurelia-open-id-connect` plugin, the latter is a wrapper over the former; thus,  while you can use [oidc-client-js](http://docs.identityserver.io/en/latest/quickstarts/6_javascript_client.html), I recommend not to do so, because then you have to write a bunch of code to match  that library with Aurelia routing system. Well, now we must create another project to handle our client-side application, Aurelia.Client, it is up to you to choose between your options, CLI or etc. After you created the project at the required plugins:

```bash
npm install aurelia-open-id-connect --save
```

Now it’s time to configure the plugin, there are three default configurations [here](https://github.com/aurelia-contrib/aurelia-open-id-connect-demos/tree/master/aurelia-app-aspnet-core/ClientApp) and [here](https://github.com/aurelia-contrib/aurelia-open-id-connect-demos/tree/master/aurelia-app/src)  for **_Auth0_**, **_Azure Active Directory_**, and **_Identity Server_**. Just copy the one for Identity Server to your project and we will change some of the most important properties that will directly affect our project.


```ts
import { OpenIdConnectConfiguration } from "aurelia-open-id-connect";
import { UserManagerSettings, WebStorageStateStore } from "oidc-client";

const appHost = "https://localhost:44347";  // you aurelia application url

export default {
  loginRedirectRoute: "/login",  // if the user is not authenticated the aurelia router will route you here
  logoutRedirectRoute: "/index", // after a successful logout the aurelia router will land you here
  unauthorizedRedirectRoute: "/login", // if the user is unauthorized you must see this page
  userManagerSettings: {

    authority: "https://localhost:44345/",  // your identity server provider uri

    automaticSilentRenew: true,

    // IdentityServer4 supports OpenID Connect Session Management
    // https://openid.net/specs/openid-connect-session-1_0.html
    monitorSession: true,
    checkSessionInterval: 2000,

    //  The client or application ID that the authority issues.
    //  this uniquely identidies your app on the server
    client_id: "aurelia_web_api_client_spa",

    filterProtocolClaims: true,
    loadUserInfo: false,

    // these two properties should match the exact properties on your client definition at server
    post_logout_redirect_uri: `${appHost}/signout-oidc`,
    redirect_uri: `${appHost}/signin-oidc`,

    //  what do you expect the server to return to you, 
    //  "id_token" for identity resources and "token" for api resources
     response_type: "id_token token",


    // this should be a subset of your AllowedScopes defined on the server, you should at least provide openid
    scope: "openid aurelia_web_api",

    // number of millisecods to wait for the authorization
    // server to response to silent renew request
    silentRequestTimeout: 10000,
    silent_redirect_uri: `${appHost}/signin-oidc`,
    userStore: new WebStorageStateStore({
      prefix: "oidc",
      store: window.localStorage,
    }),
  } as UserManagerSettings,
} as OpenIdConnectConfiguration;
```

There are two parts you must configure, **UserManagerSettings**, and **OpenIdConncetConfiguration**. for the latter you must set where the user should be redirected when login, logout are successfully done, or an unauthorized access detected, [these three lines](https://gist.github.com/shahabganji/7edac38775812f3627e226904831b2e3#file-open-id-configuration-ts-L8-L10) are for that purpose. For **UserManagerSettings**, you must set:


* [**authority:**](https://gist.github.com/shahabganji/7edac38775812f3627e226904831b2e3#file-open-id-configuration-ts-L13) The URI of your Identity Server provider in our case, http://localhost:44345
* [**client_id:**](https://gist.github.com/shahabganji/7edac38775812f3627e226904831b2e3#file-open-id-configuration-ts-L24) This is your [ClientId](https://gist.github.com/shahabganji/7edac38775812f3627e226904831b2e3#file-identityserverconfiguration-cs-L26) defined on the server.

* [**redirect_uri:**](https://gist.github.com/shahabganji/7edac38775812f3627e226904831b2e3#file-open-id-configuration-ts-L31) This should also match the [RedirectUri](https://gist.github.com/shahabganji/7edac38775812f3627e226904831b2e3#file-identityserverconfiguration-cs-L32) on the server; by default, these should be your clients URI followed by signin-oidc, you can change them to what suits you though.

* [**post_logout_redirect_uri:**](https://gist.github.com/shahabganji/7edac38775812f3627e226904831b2e3#file-open-id-configuration-ts-L30) This also should match its equivalent [property](https://gist.github.com/shahabganji/7edac38775812f3627e226904831b2e3#file-identityserverconfiguration-cs-L33) on the server; by default, this is also your application URI appended by **_signout-oidc_**.

* [**response_type:**](https://gist.github.com/shahabganji/7edac38775812f3627e226904831b2e3#file-open-id-configuration-ts-L35) Indicates what kind of tokens you expect from the server, and consequently which sort of scopes you can request later on. **id_token** for identity resources and **token** for API resources.

* [**scope:**](https://gist.github.com/shahabganji/7edac38775812f3627e226904831b2e3#file-open-id-configuration-ts-L39) This property is a subset of [AllowedScopes](https://gist.github.com/shahabganji/7edac38775812f3627e226904831b2e3#file-identityserverconfiguration-cs-L36-L40) defined on the server for this client; if **response_type** does not contain **token** you cannot request for API resource, you will get an error indicating **invalid_scope**.

That being set, you are ready to define you routes, use the usual means of declaring routes in Aurelia, the only part you should know about is that henceforth you can add **roles** property to the **settings** part of each route; it takes an array of **OpenIdConnectRoles**, **Evryone**, the default value, **Anonymous**, and **Authenticated**; each of which will influence the routing in one way or another:

* **Everyone:** The name stands for itself, nothing is checked and all users can navigate to this rout.
* **Anonymous:** This is a little bit tricky, routes having this role can be seen when no user is logged in.
* **Authenticated:** Only logged in users can navigate to this page.

**PS:**  There is a default value converter, 
[openIdConnectNavigationFilter](https://github.com/aurelia-contrib/aurelia-open-id-connect/blob/master/src/open-id-connect-navigation-filter.ts),  with the `aurelia-open-id-connect` plugin, that can be applied on `router.navigation` property to filter visible routes.

```html
<template>

  <div class="container">

    <ul repeat.for="nav of router.navigation | openIdConnectNavigationFilter:user">
      <li class="${nav.isActive ? 'active' : ''}">
        <a href.bind="nav.href">
          ${nav.title}
        </a>
      </li>
    </ul>

    <hr />

    <router-view></router-view>

  </div>

</template>
```

When setting the `roles` with `Anonymous` You can still navigate to route by changing the URL, however, it will not be shown on your navigation menu. The routing code for this sample is as the following:

```ts
import { RouterConfiguration, Router } from 'aurelia-router';

import "toastr/build/toastr.css";
import "font-awesome/css/font-awesome.css";

import { autoinject } from 'aurelia-dependency-injection';
import { PLATFORM } from 'aurelia-pal';

import { User } from "oidc-client";
import { OpenIdConnect, OpenIdConnectRoles } from "aurelia-open-id-connect";

@autoinject()
export class App {

  public router: Router;
  public user: User;

  constructor(private openIdConnect: OpenIdConnect) {
    this.openIdConnect.observeUser((user: User) => this.user = user);
  }

  private configureRouter(config: RouterConfiguration, router: Router): void {

    // switch from hash (#) to slash (/) navigation
    config.options.pushState = true;

    config.title = 'Title';
    config.map([
      {
        route: '/home', name: 'home',
        moduleId: PLATFORM.moduleName('./routes/home/home'),
        nav: true, title: 'Home',
        settings: {
          roles: [OpenIdConnectRoles.Authenticated]
        }
      },
      {
        route: ['', '/index'], name: 'index',
        moduleId: PLATFORM.moduleName('./routes/home/index'),
        nav: true, title: 'Index', settings: { roles: [OpenIdConnectRoles.Everyone] }
      },
      {
        route: '/login', name: 'login',
        moduleId: PLATFORM.moduleName('./routes/auth/login'),
        nav: true, title: 'Login', settings: { roles: [ OpenIdConnectRoles.Anonymous ] }
      }
    ]);

    this.openIdConnect.configure(config);
    this.router = router;

  }
}
```

So far so good, it’s time to know how to log in and log out of the system; using `aurelia-open-id-connect` you can inject an instance of `OpenIdConnect` and use its `login` and `logout` methods, as easy as that.

```ts
import { OpenIdConnect } from 'aurelia-open-id-connect';
import { autoinject } from 'aurelia-framework';


@autoinject()
export class Login {

  constructor(private openIdConnect: OpenIdConnect) { }

  private login() {
    this.openIdConnect.login();
  }
}
```

Well, now that we have logged into the system and have an [access_token](https://gist.github.com/shahabganji/7edac38775812f3627e226904831b2e3#file-home-ts-L14) , we should be able to call secure APIs; that can be done by adding an [Authorization header](https://gist.github.com/shahabganji/7edac38775812f3627e226904831b2e3#file-home-ts-L20) with a **Bearer** token to our HTTP requests, I used `aureia-http-client` plugin to call the REST endpoints.

```ts
import { autoinject } from "aurelia-framework";
import { OpenIdConnect } from "aurelia-open-id-connect";
import { HttpClient } from "aurelia-http-client";

@autoinject()
export class Home {

  private access_token;
  private frameworks: Array<any>;

  constructor(private openIdConnect: OpenIdConnect, private httpClient: HttpClient) { }

  private async activate() {
    this.access_token = (await this.openIdConnect.getUser()).access_token;

    this.httpClient.configure(config => {
      config.withBaseUrl("https://localhost:44346/")
        .withHeader('Accept', 'application/json')
        // adds the access token, so that we can call secure apis
        .withHeader('Authorization', `Bearer ${this.access_token}`); 
    });

    return this.httpClient.get('api/secure')
      .then(response => {
        this.frameworks = response.content;
      });

  }

  private logout() {
    this.openIdConnect.logout();
  }

}
```

An instance of **HttpClient** injected, and then configured to have the `access_token` in the header of the requests, so APIs secured with `Authorize` attribute can now be accessible.

I hope you find this article useful and will be more than happy to hear from you guys, please do not hesitate to get in touch via my [Github](https://github.com/shahabganji) or [Twitter](https://twitter.com/shahabganji). Have a great day and enjoy coding. 🙂