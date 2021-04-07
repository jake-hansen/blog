---
layout: article
title: "A Robust JWT Implementation in Go"
---

Lately, I've been working on a lot of Go projects (if you haven't seen my most recent Go project, you should [go]({% post_url 2021-04-06-how-i-created-a-twitter-vaccine-bot %}) check it out). I've found that a lot of my projects require an authentication scheme of some sort so I've spent a lot of time learning the pros and cons of various implementations. JWTs are more popular than ever, and I want to show how I've implemented them in some of my projects.

This article is not very beginner friendly. If you're looking for a primer on JWTs, I suggest reading [this](https://jwt.io/introduction). You should most definitely explore some other material on JWTs on your own until you feel pretty familiar with them. After that, this article should be a good read.

## JWTs as Access Tokens
JWTs give us a lot of benefits over a traditional session based authentication scheme - the main benefit being that there are no sessions to keep track of. All the information we need in order to verify information about a user is contained within the JWT, which is verifiable thanks to cryptographic signatures. This means that there are no database lookups to perform.

Let's take a look at the claims of a basic access token. Note this example doesn't include any custom claims.

{% highlight json %}
{
    "iss": "my-app",
    "sub": "john doe",
    "aud": "consumer-id",
    "exp": 1680849511,
    "iat": 1617759511,
    "nbf": 1617759511,

}

{% endhighlight %}

### Issuing an Access Token
Access tokens obviously just don't appear out of thin air, they have to be **issued**. *Issuing* an access token is a process in which the client exchanges a form of credentials for an access token with the server. During this process, the server needs to verify the client's credentials, and only issue an access token if the user's credentials are valid.

Now, what are client credentials? They can be anything really. Traditionally, credentials are in the form of a username/password combination. You could come up with your own credential scheme if you wanted to, but usernames and passwords work perfectly for a simple app.

##### Sidenote On Credential Management
The process of managing a database of credentials is out of scope for this post. Identity management is a complicated thing and is why Identity as a Service (IDaaS) products have become so popular. Rolling your own identity management framework is not necessarily hard to do, but like most things, it's hard to do *right*. When you have to worry about things like two-factor authentication, OAuth, and secure hashing functions, it's best to leave it to the experts. Fortunately, for basic applications that will not be running in an enterprise environment, building a basic database with functionality to add, edit, and delete users is not too hard.

___

So let's say that we are able to successfully verify a user's credentials by verifying credentials that are sent to an endpoint, say `/auth`. What do we do once we've made sure the user's credentials are correct? Give them an access token, of course!

There are a couple of things to consider when issuing an access token:

1. Duration of Access Token
   
   How long should an access token be valid? Ideally, an access token should only be valid for a short period of time, say 10-15 minutes. An access token should never be valid for a long period of time in order to reduce the attack window in which a stolen token can be used. Remember, JWTs *cannot* be revoked. So, the only thing that can revoke them is time.

2. Claims
   
   What information should be in the access token? The required claims as shown in an example above should always be included. But, you can also include custom claims. Custom claims can include anything specific that your application **needs** to know during an authenticated request. Things like the user's nickname, group memberships, or even permission can be set in the access token. Any information that is not absolutely needed by the application should be left out, in order to reduce the size of the JWT. Also, a JWT should never include any claims that contain a secret value, such as the user's password. This is because a JWT can easily be decoded by anyone who has possession of it. 

### Returning an Access Token
   
How should the token be returned to the user? In the body of the request? In a cookie? Some other way? There are multiple ways to return an access token to a user, each with pros and cons. Let's take a look at the two most common ways below.
  
  * In the body of the request - This is the simplest way, as the application just sets the access token in the body of the request, usually in JSON format. The client can then parse the body for the token, and begin to use it. The benefit in this is the client has access to the token and can send it back to the server in a variety of ways (discussed below). The downside is that the client has access to the token. How can this be a downside? Simple. Never trust the client. An XSS vulnerability could easily steal the token and then impersonate the user. We'll talk about mitigation techniques later on.

  * In a cookie - In this format, the applications sets the JWT in a set-cookie header. A cookie is then automatically set in the user's browser containing the JWT. The benefit of this way is that the client has do no work in order to send the JWT back to the application for each request. The browser automatically takes care of sending the cookie to the server. Again, the downside is that the client has access to this cookie and it can therefore be stolen by a malicious script. Alternatively, the cookie can be set as HTTP-only which prevents client side scripts from reading the cookie. This protects the token from XSS vulnerabilities. However, since the cookie is automatically sent with every request, the token then is exposed to CSRF vulnerabilities. Again, we'll talk about mitigation techniques later on.

### Storing the Token

As a client, you are limited to where you can store the token based on which format the server sends the token in. If the server sends the token in an HTTP-only cookie, then the client has no option. If the server sends the token in the body of the request, then the application can pick where to store the token. Normally, the client stores the token in local storage. Now, a lot of advice says not to do this, because then the token is exposed to theft from an XSS vulnerability. Again, we'll talk about mitigation techniques later.

### Sending the Token

As a client, once again, you are limited to how you can send the token to the server based on which format the server sends the token in. If the server sends the token in an HTTP-only cookie, then the client has no option. If the server sends the token in the body of the request, then the application can send the token to the server in a variety of ways. But, it has to use a way that the server supports. One way is to pass the token by appending it to a URL in a parameter such as `/myrequest?token={access-token}`. This is not a great way to pass tokens since they are then exposed in the URL. A better (and standard) way is to pass the token in the authorization header. This would look like `Authorization: Bearer {access-token}`. Keep in mind that the server decides where to parse the token from. The client can't just pass the token to the server in an arbitrary fashion. It has to send the token to the server in the place in which the server is expecting it.

### Design

Now that we've discussed all the possibilities, let's discuss how I decided to implement access tokens. First, the structure of the custom claims for the access token is different depending on the application I'm using them in. Some applications might need one piece of information about the user and some applications might need another. So custom claims are out of scope for my JWT package. Next, I decided to pass the access token to the client in the body of the request. This allows for great flexibility for supporting a variety of clients. Browser clients will grab the token and store it in local storage. I also decided to accept the access token in the `Authorization` header.

Here is a quick table that provides an easy view of the pros and cons, summarized from above.

| Store JWT In       | Vulnerable to XSS | Vulnerable to CSRF | Send in Auth Header | Send as Cookie |
| -------------------| :---------------: | :----------------: | :-----------------: | :------------: |
| Cookie             |                   |          X         |                     |        X       |
| HTTP-Only Cookie   |                   |          X         |                     |        X       |
| Local Storage      |         X         |                    |           X         |                |

## The Token Expiration Problem

One problem with using JWTs as access tokens is that they have a short expiration. This leads to a bad experience for the user since they would need to reexchange their credentials for a new access token every time their current access token expires. Imagine needing to re-login to a site every 10 minutes. That would be a terrible experience.

In order to fix this problem, we need a way to issue new access tokens to a user without requiring them to reenter their credentials.

### Refresh Tokens

Refresh tokens can be used *in place* of credentials to obtain a new access token. The way this works is fairly clever. When a user first logs in using their credentials, they are issued a normal access token, *and* a refresh token. A refresh token is nothing more than a JWT, but with a much longer expiry time. Something like months.

When a user's access token expires, they can obtain a new one by sending a request to an endpoint such as `/auth/refresh` containing their refresh token. As long as their refresh token is valid, the server issues the user a new access token. This process is repeated every time the user's access token expires - up until their refresh token expires. Once the user's refresh token expires, they will be required to get a new access token and refresh token by providing their credentials at the `/auth` endpoint.

Wait a minute. Isn't having a long expiration time for a JWT bad? Yes, but refresh tokens are implemented a little differently when compared to access tokens. Let me explain.

When talking about the differences in implementation between access tokens and refresh tokens, we need to discuss the criteria which is considered when determining a valid token. These criteria are different for access tokens and refresh tokens.

Access Token Validity Criteria
* Signed by application
* Not expired
* Being used after `nbf` time

Refresh Token Validity Criteria
* Signed by application
* Not expired
* Being used after `nbf` time
* *Not revoked*

Notice that the only difference in criteria between access and refresh tokens is the revocation status of the refresh token. But how do we determine revocation status of a refresh token if JWTs cannot be revoked?

Well, we cheat a little by actually storing the issued refresh token in a database along with a boolean value that determines the revocation status. So, upon each usage of the refresh token, we check all the normal criteria for a JWT, but we also look the refresh token up in the database to ensure that it hasn't been revoked. This revocation lookup is the key principle which allows us to safely set a long expiry time for a refresh token - because we then always have a 'kill switch' which can revoke access to the refresh token, instead of just relying upon the expiry of the token.

But, doesn't this take away the main advantage of JWTs in that we don't have to perform database lookups like we have to do for sessions? Yes. But the key is that we are only performing a database lookup every time the refresh token is used, which is much less frequently than the period of access token usage. So, like everything, it's a trade off.

### Issuing a Refresh Token

Refresh tokens should be issued at the same time when a user's credentials are exchanged for an access token. A refresh token usually doesn't have as many custom claims as an access token, since the refresh token only needs to identify the user.

### Returning an Refresh Token

Technically, you can return a refresh token to a user in the same ways in which you return an access token. But, refresh tokens are extremely powerful. So, we should choose a very secure method. Our best bet is to go with an HTTP-only cookie. That way, the client has no access to the refresh token making it impervious to XSS attacks. We will look at how to mitigate CSRF for the refresh token in a minute.

### Storing and Sending the Token

Since refresh tokens should only be set in an HTTP-only cookie, we don't have to worry about storing or sending the token, since the browser will take care of this for us.

### Refresh Token Replay Attacks

### CSRF Protection

In order to protect the refresh token against CSRF attacks, we need to rely on the trusty CSRF token. Remember, a CSRF token is nothing more than a random string with good entropy and is non predictable. An easy and secure way to do this is set the CSRF token as a claim in the refresh token and send the CSRF token in the body of the request when a user receives a refresh token. Whenever the user sends a refresh request, they will need to attach the CSRF token to a predetermined header. The server can then verify that the given CSRF token in the header matches the CSRF token in the refresh token's claims. 


### Design


