# AWS Lambda Authorizers (aka "Custom Authorizers")

AWS Lambda Authorizers serve as a gatekeeper to one or more of your lambda functions. They are
[one of the ways](https://youtu.be/VZqG7HjT2AQ?t=1700) AWS provides users (NEW:
[HTTP API](https://aws.amazon.com/blogs/compute/announcing-http-apis-for-amazon-api-gateway/)) to
authorize calls to your lambda via API Gateway.

## Actors

| User/Visit | App | λ Authen | λ Author | Gateway | λ Func | DB/Store | Github |
| ---------- | --- | -------- | -------- | ------- | ------ | -------- | ------ |
| Human UX/I | Up  | Servrlss | Servrlss | Up      | Srvrls | Srvrls   | 3rd pr |

The basic lifecycle of an authorized request goes like this:

## User Journey

There's a nice [YouTube Presentation from the AWS team](https://youtu.be/VZqG7HjT2AQ?t=1690) about
much of this.

### Authentication Task

```
| User/Visit | App | λ Authen | λ Author | Gateway | λ Func | DB/Store | Github |
      |---(1)-->|         |         |         |         |         |         |
      |         |----------------------------(2)--------------------------->|
      |<--------------------------------(3)---------------------------------|
      |---(4)-->|         |         |         |         |         |         |
      |         |---(5)-->|         |         |         |         |         |
      |         |         |-------------(6)------------>|         |         |
      |         |         |         |         |         |---(7)-->|         |
      |         |         |         |         |         |<--(8)---|         |
      |         |         |<------------(9)-------------|         |         |
      |         |<--(10)--|         |         |         |         |         |
      .         .         .         .         .         .         .         .
```

Authentication Steps

1. Someone visits the App (unauthenticated) and clicks "login" (Apex Up web app)
2. Link URL (TODO: alert SW): The login is a link(`<a href="github.com/auth">`) directs the visitor
   to login via a third-party (OAuth 2.0) page (e.g. Github)
3. The visitor provides their creds to the third party page/component
4. Redirect URL/auth: A redirect link (registered during App registration with the third
   party/Github) sends the user - with OAuth `code` in URL - back to the app (special route to
   trigger step 5) TODO: on DOM load, if `/auth` grab URL and `history.replaceState` to clear info
   from URL
5. HTTP POST: The app parses the URL for code and with it calls the Authentication lambda
6. HTTP POST (nested): The authenticator lambda calls the Functional lambda for handling DB xforms
7. The function finds the user or creates the user in the DB (FaunaDB) with the OAuth Token payload
   and [creates a DB secret](https://gist.github.com/colllin/fd7a40bb4f0f16603e68db0e6621369f) for
   the user for some scope of access.
8. Then retuns access & refresh tokens as a JWT to the Function
9. HTTP Response (nested): The DB/store returns the
   [JWT and Refresh Token](https://nodejs.org/dist/latest-v8.x/docs/api/http.html#http_response_setheader_name_value)
   to the Authentication lambda
10. HTTP Response: The Authentication lambda packages the secret into a JWT auth header for the user
    to use in the app (access & refresh tokens hydrated in a
    [`HttpOnly`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies) cookie)
11. TODO: (SW) Consider ServiceWorker broker for redirection back to route left in (1)

Reading Materials:

- Best Practices for handling JWTs
  [Blog from Hasura](https://hasura.io/blog/best-practices-of-using-jwt-with-graphql/)
- Refresh Tokens for FaunaDB
  [Gist](https://gist.github.com/colllin/fd7a40bb4f0f16603e68db0e6621369f)

### Authorization Task

```
| User/Visit | App | λ Authen | λ Author | Gateway | λ Func | DB/Store | Github |
      .         .         .         .         .         .         .         .
      |---(1)-->|         |         |         |         |         |         |
      |         |-------------(2)------------>|         |         |         |
      |         |         |         |     ?<-(3)->?     |         |         |
      .         .         .         .         .         .         .         .
```

Authorization Steps

1.  Authenticated users acts on a resource (some CRUD operation on DB) that resides behind AWS
    Gateway
2.  App sends the token within the HTTP Header to AWS Gateway (Apex Up Microservice)
3.  The API Gateway checks it's local cache to see if there's a known valid AWS IAM (user access)
    policy for the token.

#### If there's no valid policy is found in cache...

```
| User/Visit | App | λ Authen | λ Author | Gateway | λ Func | DB/Store | Github |
      .         .         .         .         .         .         .         .
      |         |         |         |<--(4)---|         |         |         |
      |         |         |         |--------(5)------->|         |         |
      |         |         |         |         |         |---(6)-->|         |
      |         |         |         |         |         |<--(7)---|         |
      |         |         |         |<-------(8)--------|         |         |
      |         |         |         |---(9)-->|         |         |         |
      |         |         |         |         |--(10)-->|         |         |
      |         |         |         |         |         |--(11)-->|         |
      |         |         |         |         |         |<--(12)--|         |
      |         |         |         |         |<--(13)--|         |         |
      |         |<------------(14)------------|         |         |         |

```

4. Gateway invokes the Authorizer λ (Apex Up)
5. Authorizer calls the Functional Lambda
6. Function queries the user DB to check if the token is valid and - if so - what access they have
7. Permissions are returned to the function
8. Now, the access scope is sent with the IAM policy back to the Authorizer lambda (in the HTTP
   GET/POST/etc. header)
9. The Authorizer send the IAM policy back to gateway (which - autonomously - validates that they
   have access for the operation requested)
10. If valid, gateway allows the operation via the Functional Lambda
11. That lambda can now operate on the store/DB (FaunaDB)
12. The operation returns from the DB to Function
13. Function to Gateway
14. Gateway to app state: updated (fwew! 😓)

#### If a valid policy is found in cache...

```
| User/Visit | App | λ Authen | λ Author | Gateway | λ Func | DB/Store | Github |
      .         .         .         .         .         .         .         .
      |         |         |         |         |---(4)-->|         |         |
      |         |         |         |         |         |---(5)-->|         |
      |         |         |         |         |         |<--(6)---|         |
      |         |         |         |         |<--(7)---|         |         |
      |         |<------------(8)-------------|         |         |         |

```

4. Gateway allows the operation via the Functional Lambda
5. That lambda can now operate on the store/DB (FaunaDB)
6. The operation returns from the DB
7. Function to Gateway
8. Gateway to app state: updated (fwew! 😓)

### Refresh Process

Once the short-lived access token expires the refresh token must be used to generate a new access
token

Find some guidance for implementing token refreshments
[in this gist](https://gist.github.com/colllin/fd7a40bb4f0f16603e68db0e6621369f)

```
| User/Visit | App | λ Authen | λ Author | Gateway | λ Func | DB/Store | Github |
      .         .         .         .         .         .         .         .
      |---(1)-->|         |         |         |         |         |         |
      |         |-------------(2)------------>|         |         |         |
      |         |         |         |<--(3)---|         |         |         |
      |         |         |         |--------(4)------->|         |         |
      |         |         |         |         |         |---(5)-->|         |
      |         |         |         |         |         |<--(6)---|         |
      |         |         |         |<-------(7)--------|         |         |
      |         |         |         |---(8)-->|         |         |         |
      |         |<------------(9)-------------|         |         |         |
      .         .         .         .         .         .         .         .
```

1. User tries to act on a resource
2. App sends request to Gateway, but gateway doesn't have a cached/non-expired policy for the token
3. Gateway doesn't find a cached IAM Policy that's valid (expired)
4. Authorizer notices that the JWT has expired

### Policy "Blueprint"

```js
let policy = new AuthPolicy("userIdentifier", "XXXXXXXXXXXX", apiOptions)
policy.allowMethod(AuthPolicy.HttpVerb.POST, "/locations/*")
policy.allowMethod(AuthPolicy.HttpVerb.DELETE, "/locations/*")
callback(null, policy.getPolicy())
```

Find an example
[blueprint here](https://github.com/awslabs/aws-apigateway-lambda-authorizer-blueprints/blob/master/blueprints/nodejs/index.js)

### Example IAM Policy Details

```js
{
  "Version": "2012-10-17",
  // any operation except POST
  "Statement": [
    {
      "Action": "execute-api:Invoke",
      "Effect": "Allow",
      "Resource": "arn:aws:execute-api:*:*:ff5h8tpwfh/*"
    },
    {
      "Action": "execute-api:Invoke",
      // Deny's take precedence over anything else in IAM
      "Effect": "Deny",
      "Resource": "arn:aws:execute-api:*:*:ff5h8tpwfh/*/POST/locations/*"
    }
  ]
}
```

## Development Journey

```diff

| Test/Visit | App | λ Authen | λ Author | Gateway | λ Func | DB/Store | Github |
      |         |         |         |         |         |         |         |
      |         |         |         |         |         |         |         |
      |         |         |         |         |         |         |         |
      |         |         |         |         |         |         |         |
      |         |         |         |         |         |         |         |

```
