---
title: OpenID Connect
redirect_from:
  - /openid-connect/
---

# OpenID Connect developer guide

OpenID Connect is a simple identity layer built on top of the OAuth 2.0 protocol. login.gov supports [version 1.0](http://openid.net/developers/specs) of the specification and conforms to the [iGov Profile](https://openid.net/wg/igov).

{% include basic-auth-warn.html %}

### Contents

<div markdown="1" class="compact-list">
- [Getting started](#getting-started)
- [Authorization](#authorization)
  - [Authorization response](#authorization-response)
- [Token](#token)
  - [Token response](#token-response)
- [User info](#user-info)
  - [User info response](#user-info-response)
- [Certificates](#certificates)
- [Logout](#logout)
  - [Logout response](#logout-response)
- [Example apps](#example-apps)
</div>

## Getting started

### Choosing an authentication method

login.gov supports two ways to authenticate clients: **private_key_jwt** and **PKCE**.

- **private_key_jwt** (preferred for **web apps**)
  The client sends a [JWT][jwt] signed with a private key when requesting access tokens. The corresponding public key is registered with the IdP ahead of time, similar to SAML.

- **PKCE** (preferred for **native mobile apps**)
  Short for [Proof Key for Code Exchange by OAuth Public Clients](https://tools.ietf.org/html/rfc7636) and pronounced "pixy", for this method the client sends a public identifier as well as a hashed random value generated on the client.

### Auto-discovery

Consistent with the specification, login.gov provides a JSON endpoint with data for OpenID Connect auto-discovery at:
`/.well-known/openid-configuration`

In our agency integration environment, this is available at [https://idp.int.login.gov/.well-known/openid-configuration](https://idp.int.login.gov/.well-known/openid-configuration)

## Authorization

The authorization endpoint handles authentication and authorization of a user. To present the login.gov authorization page to a user, direct them to the `/openid_connect/authorize` endpoint with the following parameters:

* **acr_values**
  Space-separated Authentication Context Class Reference values, used to specify the LOA (level of assurance) of an account, either LOA1 or LOA3. This and the `scope` determine which [user attributes]({{ site.baseurl }}/attributes) will be available in the [user unfo response](#user-info-response). The possible parameter values are:
    - `http://idmanagement.gov/ns/assurance/loa/1`
    - `http://idmanagement.gov/ns/assurance/loa/3`

* **client_id**
  Unique identifier for the client. This will be registered with the login.gov IdP in advance.

* **code_challenge** — *required for PKCE*
  The URL-safe base64 encoding of the SHA256 digest of a random value generated on the client. The original random value is referred to as the [`code_verifier`](#token-code-verifier) is used later in the token endpoint. Generating these values in Ruby could look like, for example:
  ```ruby
  code_verifier = SecureRandom.hex
  => "7a5e819dd39f17242fdeeba0c1c80be6"
  code_challenge = Digest::SHA256.base64digest(code_verifier)
  => "TdzfmaWefbtaI0Wdo6lrZCXpLu1WpamnSoSHfDUiL7Y="
  ```

* **code_challenge_method** — *required for PKCE*
  Must be `S256`, the only PKCE code challenge method we support.

* **prompt** — *optional*
  This can be either `select_account` (default behavior) or `login` (force a re-authorization even if a current IdP session is active).

* **response_type**
  Must be `code`.

* **redirect_uri**
  URI that login.gov will redirect to after a successful authorization.

* **scope**
  Space-separated string of the scopes being requested. The authorization page will display the list of attributes being requested from the user. Applications should aim to request the fewest [user attributes]({{ site.baseurl }}/attributes) and smallest scope needed. Possible values are: `openid`, `address`, `email`, `phone`, `profile:birthdate`, `profile:name`, `profile`, `social_security_number`

* **state**
  Unique value at least 32 characters long, to be returned on a successful authorization.

* **nonce**
  Unique value at least 32 characters long which will be embedded into the `id_token`. It is recommended that clients assert this value to identify any tampering.

View an example for...<span class="space"></span><button data-example="private_key_jwt">private_key_jwt</button><button data-example="pkce">PKCE</button>

<div markdown="1" data-example="private_key_jwt">
```bash
https://idp.int.login.gov/openid_connect/authorize?
  acr_values=http%3A%2F%2Fidmanagement.gov%2Fns%2Fassurance%2Floa%2F1&
  client_id=${CLIENT_ID}&
  nonce=${NONCE}&
  prompt=select_account&
  redirect_uri=${REDIRECT_URI}&
  response_type=code&
  scope=openid+email&
  state=abcdefghijklmnopabcdefghijklmnop
```
</div>
<div markdown="1" data-example="pkce" hidden="true">
```bash
https://idp.int.login.gov/openid_connect/authorize?
  acr_values=http%3A%2F%2Fidmanagement.gov%2Fns%2Fassurance%2Floa%2F1&
  client_id=${CLIENT_ID}&
  code_challenge=${CODE_CHALLENGE}&
  code_challenge_method=S256&
  nonce=${NONCE}&
  prompt=select_account&
  redirect_uri=${REDIRECT_URI}&
  response_type=code&
  scope=openid+email&
  state=abcdefghijklmnopabcdefghijklmnop
```
</div>

### Authorization response

After an authorization, login.gov will redirect to the provided `redirect_uri`.

In a **successful authorization**, the URI will contain the two parameters `code` and `state`:

- **code** — A unique authorization code the client can pass to the [token endpoint](#token).
- **state** — The `state` value originally provided by the client.

For example:

```bash
https://agency.gov/response?
  code=12345&
  state=abcdefghijklmnopabcdefghijklmnop
```

In an **unsuccessful authorization**, the URI will contain the parameters `error` and `state`, and optionally `error_description`:

- **error** — The error type, either:
  - `access_denied` — The user has either cancelled or declined to authorize the client.
  - `invalid_request` — The authorization request was invalid. See the `error_description` parameter for more details.
- **error_description** — A description of the error.
- **state** — The `state` value originally provided by the client.

For example:

```bash
https://agency.gov/response?
  error=access_denied&
  state=abcdefghijklmnopabcdefghijklmnop
```

## Token

Clients use the token endpoint to exchange the authorization `code` for an `id_token` as well as an `access_token`. To request a token, send a HTTP POST request to the `/api/openid_connect/token` URL with the following parameters:

* **client_assertion** — *required for private_key_jwt*
  A signed [JWT][jwt] with the following claims. Must be signed with the client's private key.
    * **iss** (string) — Issuer, the client's `client_id`.
    * **sub** (string) — Subject, the client's `client_id`.
    * **aud** (string) — Audience, the URL of the token endpoint. Should be `https://idp.int.login.gov/api/openid_connect/token`.
    * **jti** (string) — An un-guessable, random string generated by the client.
    * **exp** (number) — An integer timestamp of the expiration of this token (number of seconds since the Unix Epoch), should be a short period of time in the future (such as 5 minutes from now).

* **client_assertion_type** — *required for private_key_jwt*
  When using private_key_jwt, must be `urn:ietf:params:oauth:client-assertion-type:jwt-bearer`

* **code** — *required*
  The URL parameter value from the `redirect_uri` in the authorization step.

* **code_verifier** — *required for PKCE*
  The original value (before the SHA256) generated for the authorization request for PKCE that corresponds to the [`code_challenge`](#authorize-code-challenge).

* **grant_type** — *required*
  Must be `authorization_code`

View example an for...<span class="space"></span><button data-example="private_key_jwt">private_key_jwt</button><button data-example="pkce">PKCE</button>

<div markdown="1" data-example="private_key_jwt">
```bash
POST https://idp.int.login.gov/api/openid_connect/token

client_assertion=${CLIENT_ASSERTION}&
client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer&
code=${CODE}&
grant_type=authorization_code
```
</div>
<div markdown="1" data-example="pkce" hidden="true">
```bash
POST https://idp.int.login.gov/api/openid_connect/token

code=${CODE}&
code_verifier=${CODE_VERIFIER}&
grant_type=authorization_code
```
</div>

### Token response

The token response will be a JSON object containing the following:

* **access_token** (string)
  An opaque token used to authenticate to the [user info endpoint](#user-info).

* **token_type** (string)
  Describes the kind of access token. Will always be `Bearer`.

* **expires_in** (number)
  The number of seconds that the access token will expire in.

* **id_token** (string)
  A signed [JWT][jwt] that contains basic attributes about the user such as user ID for this client (encoded as the `sub` claim) as well as the claims requested as part of the `scope` in the authorization request. See the [User Info Response](#user-info-response) section for details on the claims. The public key to verify this JWT is available from the [certs](#certs) endpoint.

Here's an example token response:

```json
{
  "access_token": "hhJES3wcgjI55jzjBvZpNQ",
  "token_type": "Bearer",
  "expires_in": 3600,
  "id_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJiMmQyZDExNS0xZDdlLTQ1NzktYjlkNi1mOGU4NGY0ZjU2Y2EiLCJpc3MiOiJodHRwczovL2lkcC5pbnQubG9naW4uZ292IiwiYWNyIjoiaHR0cDovL2lkbWFuYWdlbWVudC5nb3YvbnMvYXNzdXJhbmNlL2xvYS8xIiwibm9uY2UiOiJhYWQwYWE5NjljMTU2YjJkZmE2ODVmODg1ZmFjNzA4MyIsImF1ZCI6InVybjpnb3Y6Z3NhOm9wZW5pZGNvbm5lY3Q6ZGV2ZWxvcG1lbnQiLCJqdGkiOiJqQzdOblU4ZE5OVjVsaXNRQm0xanRBIiwiYXRfaGFzaCI6InRsTmJpcXIxTHIyWWNOUkdqendsSWciLCJjX2hhc2giOiJoWGpxN2tPcnRRS196YV82dE9OeGN3IiwiZXhwIjoxNDg5Njk0MTk2LCJpYXQiOjE0ODk2OTQxOTgsIm5iZiI6MTQ4OTY5NDE5OH0.pVbPF-2LJSG1fE9thn27PwmDlNdlc3mEm7fFxb8ZADdRvYmDMnDPuZ3TGHl0ttK78H8NH7rBpH85LZzRNtCcWjS7QcycXHMn00Cuq_Bpbn7NRdf3ktxkBrpqyzIArLezVJJVXn2EeykXMvzlO-fJ7CaDUaJMqkDhKOK6caRYePBLbZJFl0Ri25bqXugguAYTyX9HACaxMNFtQOwmUCVVr6WYL1AMV5WmaswZtdE8POxYdhzwj777rkgSg555GoBDZy3MetapbT0csSWqVJ13skWTXBRrOiQQ70wzHAu_3ktBDXNoLx4kG1fr1BiMEbHjKsHs14X8LCBcIMdt49hIZg"
}
```

The **id_token** contains the following claims:

* **acr** (string) — Authentication Context Class Reference value or LOA (level of authentication) of the returned claims, from the original [authorization request](#authorization-request).
* **at_hash** (string) — Access token hash, a url-safe base-64 encoding of the left 128 bits of the SHA256 of the `access_token` value. Provided so the client can verify the `access_token` value.
* **aud** (string) — Audience, the client ID.
* **c_hash** (string) — Code hash, a url-safe base-64 encoding of the left 128 bits of the SHA256 of the authorization `code` value. Provided so the client verify the `code` value.
* **exp** (number) — Expiration, an integer timestamp of the expiration of this token (number of seconds since the Unix Epoch).
* **iat** (number) — Issued at, an integer timestamp of when the token was created (number of seconds since the Unix Epoch).
* **iss** (string) — Issuer, will be `https://idp.int.login.gov`.
* **jti** (string) — An random string generated to ensure uniqueness.
* **nbf** (number) — "Not before", an integer timestamp of when the token will start to be valid (number of seconds since the Unix Epoch).
* **nonce** (string) — The nonce provided by the client in the [authorization request](#authorization-request)
* **sub** (string) — Subject, unique ID for this user. This ID is unique per client.

Here's an example decoded **id_token**:

```json
{
  "sub": "b2d2d115-1d7e-4579-b9d6-f8e84f4f56ca",
  "iss": "https://idp.int.login.gov",
  "acr": "http://idmanagement.gov/ns/assurance/loa/1",
  "nonce": "aad0aa969c156b2dfa685f885fac7083",
  "aud": "urn:gov:gsa:openidconnect:development",
  "jti": "jC7NnU8dNNV5lisQBm1jtA",
  "at_hash": "tlNbiqr1Lr2YcNRGjzwlIg",
  "c_hash": "hXjq7kOrtQK_za_6tONxcw",
  "exp": 1489694196,
  "iat": 1489694198,
  "nbf": 1489694198
}
```

## User info

The userinfo endpoint is used to retrieve [user attributes]({{ site.baseurl }}/attributes). Clients use the `access_token` from the [token response](#token-response) as a bearer token in the [HTTP Authorization header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization). Send an HTTP GET request to the `/api/openid_connect/userinfo` endpoint, for example:

```
GET https://idp.int.login.gov/api/openid_connect/userinfo
Authorization: Bearer hhJES3wcgjI55jzjBvZpNQ
```

### User info response

Note, login.gov supports some of the [standard claims](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims) from OpenID Connect 1.0.

* **address** (object)
  A JSON object, per the OpenID Connect 1.0 spec [Address Claim](https://openid.net/specs/openid-connect-core-1_0.html#AddressClaim)
  Requires the `address` scope and an LOA 3 account.

* **birthdate** (string)
  Birthdate, formatted as ISO 8601:2004, that is `YYYY-MM-DD`
  Requires `profile` or `profile:birthdate` scopes and an LOA 3 account.

* **email** (string)
  The user's email.
  Requires the `email` scope.

* **email_verified** (boolean)
  Whether or not the `email` has been verified. Currently, login.gov only supports verified emails.
  Requires the `email` scope.

* **family_name** (string)
  The user's last (family) name.
  Requires `profile` or `profile:name` scopes and an LOA 3 account.

* **given_name** (string)
  The user's first (given) name.
  Requires `profile` or `profile:name` scopes and an LOA 3 account.

* **iss** (string)
  Issuer, will be the IdP's URL, for example `https://idp.int.login.gov` when testing in login.gov's agency integration environment.

* **phone** (string)
  User's phone number, formatted as E.164, for example: `+1 (555) 555-5555`
  Requires the `phone` scope and an LOA 3 account.

* **phone_verified** (boolean)
  Whether or not the `phone` has been verified. Currently, login.gov only supports verified phones.
  Requires the `phone` scope and an LOA 3 account.

* **social_security_number** (string)
  User's Social Security number.
  Requires the `social_security_number` scope and an LOA 3 account.

* **sub** (string)
  The subject, the UUID for this user. This is current unique *per client*, although will be transitioned to *per agency* eventually.

Here's an example response:

```json
{
  "address": {
    "formatted": "123 Main St Apt 123\nWashington, DC 20001",
    "street_address": "123 Main St Apt 123",
    "locality": "Washington",
    "region": "DC",
    "postal_code": "20001"
  },
  "birthdate": "1970-01-01",
  "email": "test@example.com",
  "email_verified": true,
  "family_name": "Smith",
  "given_name": "John",
  "iss": "https://idp.int.login.gov",
  "phone": "+1 (555) 555-5555",
  "phone_verified": true,
  "social_security_number": "111223333",
  "sub": "b2d2d115-1d7e-4579-b9d6-f8e84f4f56ca"
}
```

## Certificates

The public key to verify signed JWTs from login.gov (such as the `id_token`) is available in [JWK](https://tools.ietf.org/html/rfc7517) format at the `/api/openid_connect/certs` endpoint, for example in the agency integration environment at [https://idp.int.login.gov/api/openid_connect/certs](https://idp.int.login.gov/api/openid_connect/certs)

## Logout

login.gov supports [RP-Initiated Logout](https://openid.net/specs/openid-connect-session-1_0.html#RPLogout). Clients can direct users to the `/openid_connect/logout` endpoint with the following parameters to log them out of their current login.gov session and redirect back to to the client:

- **id_token_hint**
  An `id_token` value from the [token endpoint response](#token-response).

- **post_logout_redirect_uri**
  The URI login.gov will redirect to after logout.

- **state**
  Unique value at least 32 characters long, to be returned on a successful logout.

Here's an example logout request:

```bash
https://idp.int.login.gov/openid_connect/logout?
  id_token_hint=eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJiMmQyZDExNS0xZDdlLTQ1NzktYjlkNi1mOGU4NGY0ZjU2Y2EiLCJpc3MiOiJodHRwczovL2lkcC5pbnQubG9naW4uZ292IiwiYWNyIjoiaHR0cDovL2lkbWFuYWdlbWVudC5nb3YvbnMvYXNzdXJhbmNlL2xvYS8xIiwibm9uY2UiOiJhYWQwYWE5NjljMTU2YjJkZmE2ODVmODg1ZmFjNzA4MyIsImF1ZCI6InVybjpnb3Y6Z3NhOm9wZW5pZGNvbm5lY3Q6ZGV2ZWxvcG1lbnQiLCJqdGkiOiJqQzdOblU4ZE5OVjVsaXNRQm0xanRBIiwiYXRfaGFzaCI6InRsTmJpcXIxTHIyWWNOUkdqendsSWciLCJjX2hhc2giOiJoWGpxN2tPcnRRS196YV82dE9OeGN3IiwiZXhwIjoxNDg5Njk0MTk2LCJpYXQiOjE0ODk2OTQxOTgsIm5iZiI6MTQ4OTY5NDE5OH0.pVbPF-2LJSG1fE9thn27PwmDlNdlc3mEm7fFxb8ZADdRvYmDMnDPuZ3TGHl0ttK78H8NH7rBpH85LZzRNtCcWjS7QcycXHMn00Cuq_Bpbn7NRdf3ktxkBrpqyzIArLezVJJVXn2EeykXMvzlO-fJ7CaDUaJMqkDhKOK6caRYePBLbZJFl0Ri25bqXugguAYTyX9HACaxMNFtQOwmUCVVr6WYL1AMV5WmaswZtdE8POxYdhzwj777rkgSg555GoBDZy3MetapbT0csSWqVJ13skWTXBRrOiQQ70wzHAu_3ktBDXNoLx4kG1fr1BiMEbHjKsHs14X8LCBcIMdt49hIZg&
  post_logout_redirect_uri=${REDIRECT_URI}&
  state=abcdefghijklmnopabcdefghijklmnop
```

### Logout response

After an authorization, login.gov will redirect to the provided `redirect_uri` with additional URL query parameters added.

On a **successful logout**, the URI will contain the `state` parameter originally provided by the client in the logout request.

Here's an example logout response:

```bash
https://example.com/response?
  state=abcdefghijklmnopabcdefghijklmnop
```

## Example apps

The login.gov team has created example clients to speed up your development, all open source in the public domain.

- [C# / ASP.NET](https://github.com/18F/identity-openidconnect-aspnet)
- [Java / Spring Security](https://github.com/18F/identity-oidc-java-spring-security)
- [Java / Spring Boot](https://github.com/18F/identity-oidc-java-spring-boot)
- [Java / Spring Boot XML](https://github.com/18F/identity-oidc-java-spring-boot-xml)
- [iOS (Swift) / AppAuth](https://github.com/18F/identity-openidconnect-ios-client)
- [Ruby / Sinatra](https://github.com/18F/identity-openidconnect-sinatra)
- [Node.js / Express.js](https://github.com/18F/identity-oidc-expressjs)
  <!-- Also: https://github.com/18F/identity-oidc-nodejs-express -->
- [Groovy](https://github.com/18F/identity-oidc-groovy)
- [Python / Django](https://github.com/18F/identity-oidc-python-django)


[jwt]: https://jwt.io/


<script type="text/javascript">
  function showExamples(type) {
    Array.prototype.slice.call(document.querySelectorAll('button[data-example]')).forEach(function(button) {
      var show = button.getAttribute('data-example') == type;
      button.className = show ? 'usa-button' : 'usa-button-secondary';
    });

    Array.prototype.slice.call(document.querySelectorAll('div[data-example]')).forEach(function(example) {
      var show = example.getAttribute('data-example') == type;
      if (show) {
        example.removeAttribute('hidden');
      } else {
        example.setAttribute('hidden', 'true');
      }
    });
  }

  Array.prototype.slice.call(document.querySelectorAll('button[data-example]')).forEach(function(button) {
    button.onclick = function() {
      showExamples(this.getAttribute('data-example'));
    };
  });

  showExamples('private_key_jwt');
</script>
