title: Generate/Validate JSON Web Tokens(JWT) in ASP.NET Core
date: August 17 2020
category: aurelia
tags:
    - aspnetcore
    - authorization
    - authentication
    - jwt
    - jwt-token
    - identity
    - identityserver
---

Last week there was a requirement within one of our backend teams to generate and validate JWT without using [IdentityServer4](https://identityserver4.readthedocs.io/en/latest/). I vividly remember that last year I have seen such same requirement in another company and they have manually done so many steps to do so and it was a painful process for them to achieve that goal. Here in this article we will discuss how one might have such feature in their API using built-in classes and functionalities in ASP.NET Core.

<!-- more -->

## What is **JWT**

JSON Web Token (JWT) is an Internet standard for creating data with optional signature and/or optional encryption whose payload holds JSON that asserts some number of claims. The key can be signed using various mechanisms, with a secret key, or a public/private key pair. Then later on these mechanism will help to validate the token and make sure that it is what it claims. For more information regarding JWT see [here](https://jwt.io/introduction/).


## Generate JWT

ASP.NET Core provides classes to facilitate and secure the way developers intend to generate or validate JWTs. One of the main participants in this process is `JwtSecurityTokenHandler` class; It's responsibility is to create tokens based on `JwtSecurityToken` or a description of the token by `SecurityTokenDescriptor`, in the latter the programmer describes what information should the token contain and what the signing credentials, algorithm, should be; with the `JwtSecurityToken` the developer explicitly sets headers, payload, and signing mechanism using several classes provided by the framework, for instance, `JwtHeader` and `JwtPayload` classes. Let's see that in action.

You need to install a nuget package to be able to generate or validate JWT, to do so add the following package:

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer --version 3.1.8
```

```cs
var securityKey = new SymmetricSecurityKey(
        Encoding.UTF8.GetBytes("7h!$S40u1d83@$7r0n9P@5$Word"));
var header = new JwtHeader(
        new SigningCredentials(
                securityKey,
                SecurityAlgorithms.HmacSha512Signature
        ));

var claims = new[]
{
    new Claim(ClaimTypes.NameIdentifier, "user1"),
    new Claim(ClaimTypes.Role, "admin"),
};
var payload = new JwtPayload(
                        issuer: "http://localhost:6001",
                        audience: "my-api",
                        claims: claims, null,
                        expires: DateTime.UtcNow.AddDays(7),null);

var token = new JwtSecurityToken(header, payload);

var tokenHandler = new JwtSecurityTokenHandler();

return tokenHandler.WriteToken(token);
```

Try to reach the `account/login` endpoint to test the output, either by using swagger or running the following curl command:

```bash
curl -X POST "https://localhost:6001/Account/login" -H "accept: application/json" -H "Content-Type: application/json" -d "{\"username\":\"shahab\",\"password\":\"somepassword\"}"
```

Get the out put and check it in [jwt.io](jwt.io). So far so good, now we have a valid json web token, which indicates that our user is also in the admin role. now it is time to validate the token when the user intends to run protected endpoints. Put an `[Authorize]` attribute on top of other controllers, and let's see how we could go further.

## Validate JWT

Now that we have issued a json token, it is time to pass it back to the server to be able to access protected endpoints; now, the asp.net application at the endpoint should be able to 1. validate the token and 2. grab the information, claims, hidden in that token and set them appropriately. To do so use the following code in the `ConfigureServices` of the `Startup` class:


```csharp
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(JwtBearerDefaults.AuthenticationScheme, options =>
        {
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuerSigningKey = true,
                ValidateIssuer = true,
                ValidateAudience = true,
                ValidIssuer = "http://localhost:6001",
                ValidAudience = "my-api",
                IssuerSigningKey = new SymmetricSecurityKey(
                    Encoding.UTF8.GetBytes("7h!$S40u1d83@$7r0n9P@5$Word"))
            };

            options.Events = new JwtBearerEvents
            {
                OnMessageReceived = ValidateToken
            };
        });
```

**PS:** In the [sample]() on GitHub some configuration files are placed in `appsettings.json` and will be read from there.

First we enabled authentication services in our application and then indicated that we want to use a bearer token, added some options that will be used for validating the bearer token, and then assigned a method to do the actual validation when a message is received in the pipeline.

```csharp
public static Task ValidateToken(MessageReceivedContext context)
{
    try
    {
        context.Token = GetToken(context.Request);

        var tokenHandler = new JwtSecurityTokenHandler();
        tokenHandler.ValidateToken(context.Token, context.Options.TokenValidationParameters, out var validatedToken);

        var jwtSecurityToken = validatedToken as JwtSecurityToken;

        context.Principal = new ClaimsPrincipal();

        var claimsIdentity = new ClaimsIdentity(jwtSecurityToken.Claims.ToList(),
                "JwtBearerToken", ClaimTypes.NameIdentifier, ClaimTypes.Role);
        context.Principal.AddIdentity(claimsIdentity);

        context.Success();

        return Task.CompletedTask;
    }
    catch (Exception e)
    {
        context.Fail(e);
    }

    return Task.CompletedTask;
}
```

First grab the token from the request, it could be in an `Authorization` header with a string value of `Bearer ` plus the token, or somewhere in the query string. After getting the token we used a `JwtSecurityTokenHandler` and the options defined previously, which are available through the `context` to validate the bearer token; After that we create a clain identity and set which claims to set for name and role types.

Now, you by adding an `Authorization` header, with the value of `Bearer ` plus your generated toke you could access endpoints that are secured via `Authorization` attribute.


## Conclusion

ASP.NET Core has so many built-in libraries and mechanisms to support developers for a better implementation of the functional and non-functional requirements; classes you've seen so far in this article are one of the many, just to support Json Web Tokens. I hope you find this article fruitful and enjoy coding ;)
