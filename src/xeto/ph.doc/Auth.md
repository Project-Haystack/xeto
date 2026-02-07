<!--
title:      Authentication
author:     Matthew Giannini
creation:   13 Jul 2016
copyright:  Copyright (c) 2016, Project-Haystack
license:    Licensed under the Academic Free License version 3.0
-->

# Overview
This section covers various aspects of authentication in Haystack.

# HTTP Authentication
HTTP authentication defines a general purpose, pluggable, authentication protocol
that can be used by IoT clients to authenticate themselves with a compliant
server. After successfully authenticating, the client receives an auth token
from the server that can be used to gain access to protected resources.

## Introduction
The high-level steps to obtain an auth token are:

1. Hello Handshake
  a. The client sends a hello message to the server identifying the user who wants to authenticate.
  b. The server responds indicating which authentication mechanisms are supported for that user.
2. Authentication Exchange
  a. The client picks an authentication mechanism and exchanges messages with the server to  authenticate.
  b. If the server authenticates the user, it sends the client an auth token it can use to access protected resources.
3. The client uses the auth token for all future requests to the server.

The rest of this document describes this process in more detail. We also define
a standard set of authentication mechanisms and define how the authentication
exchange should work for each mechanism.

## HTTP1/1. Authentication
This specification is based on the authentication framework presented in
"HTTP/1.1: Authentication" [RFC7235](#references). We build on this framework
by adding an explicit method for establishing what the supported
authentication mechanisms are for a particular user. There are also a few
restrictions on that specification that compliant clients and servers
must adhere to.

### Syntax restrictions
In order to make the parsing of authentication headers as easy as possible
for clients and servers, we place the following restrictions on the syntax
rules for encoding authentication headers.

- "auth-param" syntax MUST NOT use "quoted-string" syntax
- Auth schemes ("challenge" and "credentials") MUST NOT use "token68"

The following modified syntax rules MUST be used for constructing authentication
headers:

```
auth-scheme = token
auth-param  = token BWS "=" BWS token

challenge   = auth-scheme [ 1*SP #auth-param ]
credentials = auth-scheme [ 1*SP #auth-param ]
```
Essentially, we only allow the "token" syntax in any production rule.  If
a raw auth-param value is not a valid token, then the value SHOULD be
encoded using `base64url` with **no padding** [RFC4648](#references)

## Authentication Protocol
The authentication protocol is a series of HTTP requests to the server. The
URI to use for authentication is determined by the server. If a client attempts
to access a protected resource without a valid auth token, the server should
return a 401 challenge.

All messages to the server MUST use the HTTP GET method.

The server MAY include a "handshakeToken" parameter in its responses to
the client. The handshakeToken is used by the server to maintain state between
requests. If the server includes this parameter, the client MUST include it
on its next request to the server. Its value is opaque and server-defined.

### Hello
The first step in the authentication protocol is for the client to initiate
the authentication handshake with a hello message. The Authorization header
is used to supply the username of the client using the HELLO auth scheme.
The username parameter is the base64url encoding of the UTF8-encoded username.

The server responds to a hello request with a 401 (Unauthorized) response.
The response uses the WWW-Authenticate header to indicate to the client
what authentication mechanisms are supported for the supplied username.
If multiple authentication mechanisms are supported, the server SHOULD specify
them in order of most preferred to least preferred.

If the server does not recognize the username, it SHOULD still return a
401 (unauthorized) with at least one valid authentication mechanism. During the
authentication exchange, the user will not validate, and at that point a
403 (Forbidden) status code should be sent. This approach makes it more
difficult for a potential attacker to determine valid usernames for the system.

In this example, the username is “user", and the server responds indicating
that user may authenticate with SCRAM. A "handshakeToken" is included in
the response, so the client must include it on its next request.

```
C: GET /haystack/about HTTP/1.1
   Host: server.example.com
   Authorization: HELLO username=dXNlcg

S: HTTP/1.1 401 Unauthorized
   WWW-Authenticate: SCRAM hash=SHA-256, handshakeToken=aabbbcc
```

### Authentication Exchange
At this point the client may choose which authentication mechanism it wants to
use for authentication. The rest of the handshake is defined based on this
choice. The authentication mechanism MUST be defined in this document to be
compliant with this specification.

During the authentication exchange, if the client fails to authenticate then
the server MUST respond with a 403 (Forbidden) status code.

See the [Authentication Mechanisms](#authentication-mechanisms) section for details on the
supported mechanisms.

### Auth Token
Once the server has authenticated the client, the last message it sends to the
client MUST be a 200 (Ok). The response MUST use the Authentication-Info
[RFC7615](#references)
header to return an “authToken" that the client uses for
future requests to protected resources. The parameter MUST be named `authToken`.
Its value is server-defined and opaque as far as this specification is
concerned.

The server MAY also return additional parameters in the header. For example,
the last server message for SCRAM authentication contains information
the client can use to verify the server.

```
// Authentication Hello
// Authentication Exchange...
S: HTTP/1.1 200 Ok
   Authentication-Info: authToken=xxyyzz
```

The client is now authenticated with the server and may access protected
resources. All requests for protected resources MUST use the Authorization
header to pass the "authToken" to the server. The “Bearer" auth scheme MUST be
used to pass the token.

```
C: GET /some/resource HTTP/1.1
   Host: server.example.com
   Authorization: BEARER authToken=xxyyzz
   [...]
```

## Authentication Mechanisms
This section lists the supported auth schemes. A compliant client and server
must implement the authentication exchange for a particular auth scheme as
defined in this document.

All clients and servers MUST support SCRAM with SHA-256 as the hash function.

The following auth parameters are valid for all authentication schemes:

- The server MAY include a `handshakeToken` parameter in its responses to
the client. The handshakeToken is used by the server to maintain state between
requests during the authentication exchange. If the server includes this
parameter, the client MUST include it on its next request to the server. Its
value is opaque and server-defined.

# SCRAM
The `SCRAM` auth scheme SHALL be used to authenticate the client using the
Salted Challenge Response Authentication Mechanism (SCRAM).
SCRAM is defined in [RFC5802](#references).

The cryptographic hash function to use is specified as an auth parameter
in the server's response to the hello message. Regardless of which
hash function is used, the authentication
exchange messages are the same, and defined below. The approach used in this
document is based on the experimental "Salted Challenged Response HTTP
Authentication Mechanism" [RFC7804](#references).

In this example, the username is “user" and the password is “pencil". SHA-256
is used as the cryptographic hash function.

## Hello
Note: Header lines have been split to ease readability. All base64url encodings
are without padding.

The client sends a hello message indicating identifying itself as "user".

```
C: GET /haystack/about HTTP/1.1
   Host: server.example.com
   Authorization: HELLO username=dXNlcg
```

The server responds indicating that the client should authenticate with SCRAM.
- The "hash" attribute MUST be included to specify the cryptographic hash function.
  The following are supported values for the hash attribute:
  - SHA-256
  - SHA-512
- The server included a "handshakeToken", so the client MUST include it on
its next request.

```
S: HTTP/1.1 401 Unauthorized
   WWW-Authenticate: SCRAM hash=SHA-256, handshakeToken=aabbcc
```

## Authentication Exchange
The client initiates the authentication exchange.
The Authorization header specifies SCRAM as the authentication
scheme. The following attributes are specified:

- The "handshakeToken" is included since the server included one its response
to the hello message.
- The “data" attribute contains the base64url-encoded “client-first-message".

```
C: GET /haystack/about HTTP/1.1
   Host: server.example.com
   Authorization: SCRAM handshakeToken=aabbcc,
       data=biwsbj11c2VyLHI9ck9wck5HZndFYmVSV2diTkVrcU8K
```

The server responds to the client with the following attributes:

- A “handshakeToken" is specified. So the client MUST include this parameter
on its next request to the server.
- The "hash" attribute MUST be specified to indicate which hash function
is being used.
- The “data" attribute contains the base64url-encoded “server-first-message".

```
S: HTTP/1.1 401 Unauthorized
   WWW-Authenticate: SCRAM handshakeToken=authAABBCC, hash=SHA-256,
       data=cj1yT3ByTkdmd0ViZVJXZ2JORWtxTyVodllEcFdVYTJSYVRDQWZ1eEZJ
           bGopaE5sRixzPVcyMlphSjBTTlk3c29Fc1VFamI2Z1E9PSxpPTQwOTYK
```

The client continues the authentication exchange with another GET to /haystack/about.
The following attributes are specified:

- The “handshakeToken" is included since the server included one in its
previous response.
- The “data" attribute contains the base64url-encoded “client-final-message".

```
C: GET /haystack/about HTTP/1.1
   Host: server.example.com
   Authorization: SCRAM handshakeToken=authAABBCC,
       data=Yz1iaXdzLHI9ck9wck5HZndFYmVSV2diTkVrcU8laHZZRHBXVWEyUmFUQ
           0FmdXhGSWxqKWhObEYscD1kSHpiWmFwV0lrNGpVaE4rVXRlOXl0YWc5empm
           TUhnc3FtbWl6N0FuZFZRPQo
```

The final server message indicates that the user is authenticated.
Since this is the final server message, it responds with a 200 (Ok)
as described in the [Auth Token](#auth-token) section. The following
attributes are specified:

- The server includes the required "authToken"
- The "hash" attribute MUST be specified to indicate what hash function to use
when validating the server key.
- The “data" attribute contains the base64url-encoded "server-final-message"
The client SHOULD authenticate the server by checking the validity of the
server final message.

```
S: HTTP/1.1 200 Ok
   Authentication-Info: authToken=AuthenticatedTokenXXYYZZ, hash=SHA-256,
       data=dj02cnJpVFJCaTIzV3BSUi93dHVwK21NaFVaVW4vZEI1bkxUSlJzamw5NUc0PQo
```

# PLAINTEXT
The `PLAINTEXT` auth scheme SHALL be used to authenticate the client using a base64url
encoded username and password. Use of the `PLAINTEXT` scheme MUST only be
allowed over a secure TLS connection.

## Hello
Note: Header lines have been split to ease readability. All base64url encodings are
without padding. In the following example, a user named `user` with
password `password` authenticates with the server.

The client sends a hello message identifying the user as `user`.

```
C: GET /haystack/about HTTP/1.1
   HOST: server.example.com
   Authorization: HELLO username=dXNlcg
```

The server responds to the client indicating the user should authenticate using `PLAINTEXT`.

```
S: HTTP/1.1 401 Unauthorized
   WWW-Authenticate: PLAINTEXT
```

## Authentication Exchange

The client continues the authentication exchange. The following attributes are included:

- `username`: the base64url-encoded username
- `password`: the base64url-encoded password

```
C: GET /haystack/about HTTP/1.1
   Host: server.example.com
   Authorization: PLAINTEXT username=dXNlcg, password=cGFzc3dvcmQ
```

The final server message indicates the user is authenticated.

```
S: HTTP/1.1 200 Ok
   Authentication-Info: authToken=AuthTokenXXYYZZ
```

# References
 [RFC4648](https://tools.ietf.org/pdf/rfc4648.pdf): The Base16, Base32, Base64 Data Encodings

 [RFC5802](https://tools.ietf.org/html/rfc5802): Salted Challenge Response Authentication Mechanism (SCRAM) SASL and GSS-API Mechanism

 [RFC7235](https://tools.ietf.org/html/rfc7235): Hypertext Transfer Protocol (HTTP/1.1) : Authentication

 [RFC7615](https://tools.ietf.org/html/rfc7615): HTTP Authentication-Info and Proxy-Authentication-Info Response Header Fields

 [RFC7804](https://tools.ietf.org/html/rfc7804): Salted Challenge Response HTTP Authentication Mechanism
