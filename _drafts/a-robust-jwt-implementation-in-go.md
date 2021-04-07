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
   
   How long should an access token be valid? Ideally, an access token should only be valid for a short period of time, say 10-15 minutes. An access token should never be valid for a long period of time in order to reduce the attack window in which a stolen token can be used. 

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

## The Token Expiration Problem

One problem with access tokens is that they have a short expiration. 