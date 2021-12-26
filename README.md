# Pulsoid API


## Table of contents
1. [General integration information](#general_information)
2. [Authentication](#authentication)
	1. [OAuth2](#oauth2_intro)
		1. [OAuth2 Authroization Code](#oauth2_auth_code)  
		2. [OAuth2 Refreshing the token](#oauth2_refresh_token)
	3. [Manual Token issuing](#manual_token_issuing)
	4. [Validate token](#validate)
3. [Scopes](#scopes)
4. [Errors](#errorsr)
5. [Endpoints](#endpoints)
	1. [Read Heart Rate via Rest](#read_heart_rate_via_rest)
	2. [Read Heart Rate via WebSocket](#read_heart_rate_via_websocket)
	3. [Write Heart Rate via Rest](#write_heart_rate_via_rest)(not implemented)
	4. [Read user's data](#read_profile)

## General integration information<a name="general_information"></a>
Welcome to the Pulsoid integration guide. Root Pulsoid API domain is `https://dev.pulsoid.net`. All communication is done in JSON. All endpoints have a soft rate limit of 20 requests per second. 
## Authentication<a name="authentication"></a>
### OAuth2<a name="oauth2_intro"></a>
This guide describes how to use Pulsoid Authentication to enable your application to take actions on behalf of a Pulsoid account or access certain data about users’ accounts. The preferred method of authentication is OAuth. We use parts of the OAuth 2.0 protocol.
### OAuth2 implicit code flow <a name="oauth2_auth_code"></a>
[“Authorization Code Grant” in the OAuth2 RFC](https://datatracker.ietf.org/doc/html/rfc6749#section-4.1)

1) Send the user you want to authenticate to your registered redirect URI. An authorization page will ask the user to sign up or log into Pulsoid and allow the user to choose whether to authorize your application/identity system. 
   
Create a `<a href="">login</a>`:
```bash
GET https://pulsoid.net/oauth2/authorize
    ?client_id=<your client ID>
    &redirect_uri=<your registered redirect URI>
    &response_type=code
    &scope=<space-separated list of scopes>
    &state=<unique token, generated by your application>
```
Parameters explained:

| Name   |      Type      |  Description |
|----------|:-------------:|------:|
| client_id |  string | Your client ID. |
| redirect_uri |    string   |   Your registered redirect URI. This must exactly match the redirect URI registered in the prior. |
| response_type | string |    Should be always `code`|
| scope | string |    Comma-separated list of scopes. |
| state | string |    Your unique token, generated by your application. This is an OAuth 2.0 opaque value, used to avoid CSRF attacks. This value is echoed back in the response. |

In our example, you request access to read heart rate data and send the user to http://localhost:    
```bash
GET 'https://pulsoid.net/oauth2/authorize?response_type=code&client_id=3d3fa070-8358-4984-ae32-94392185df63&redirect_uri=http://localhost&scope=data:heart_rate:read&state=a52beaeb-c491-4cd3-b915-16fed71e17a8'
```
2) If the user authorizes your application, the user is redirected to your redirect URL:
```bash
https://<your registered redirect URI>/?code=<authorization code>&state=<echoed back state your application path on authorization step>
```
The OAuth 2.0 authorization code is a randomly generated string. It is used in the next step, a request made to the token endpoint in exchange for an access token.
In our example, your user gets redirected to:
```bash
http://localhost/?code=fedc8790-df28-4928-9dcf-55a4d7aa1f5e
    &state=a52beaeb-c491-4cd3-b915-16fed71e17a8
```

3) On your server, get an access token by making this request:
```bash
POST https://pulsoid.net/oauth2/token
Content-Type: application/x-www-form-urlencoded

client_id=<your client ID>
&client_secret=<your client secret>
&code=<authorization code received above>
&grant_type=authorization_code
&redirect_uri=<your registered redirect URI>
```
Here is a sample request:
```bash
POST https://pulsoid.net/oauth2/token
Content-Type: application/x-www-form-urlencoded   

grant_type=authorization_code 
&code=fedc8790-df28-4928-9dcf-55a4d7aa1f5e
&client_id=3d3fa070-8358-4984-ae32-94392185df63
&client_secret=a8262283-f568-4ec3-be84-1c4758dc1a82
&redirect_uri=http://localhost
```
4) We respond with a JSON-encoded access token. The response looks like this:
```bash
{
  "access_token": "<user access token>",
  "refresh_token": "<refresh token>",
  "expires_in": <number of seconds until the token expires>,
  "token_type": "bearer"
}
```

In our example:
```bash
{
  "access_token": "17ebb971-f558-48f2-81b1-788ea927c509",
  "refresh_token": "c6f30bc4-9a04-4e66-a1a1-080fad703a9e",
  "expires_in": 3600,
  "token_type": "bearer"
}
```

Note that code can be exchanged for access token only once.

### Refreshing access tokens<a name="oauth2_refresh_token"></a>

New OAuth2 access tokens have expirations. Token-expiration periods vary in length, based on how the token was acquired.
Tokens return an expires_in field indicating how long the token should last. However, you should build your applications
in such a way that they are resilient to token authentication failures. In other words, an application capable of
refreshing tokens should not need to know how long a token will live. Rather, it should be prepared to deal with the
token becoming invalid at any time.

To allow for applications to remain authenticated for long periods in a world of expiring tokens, we allow for sessions
to be refreshed, in accordance with the guidelines
in [“Refreshing an Access Token” in the OAuth2 RFC](https://tools.ietf.org/html/rfc6749#section-6). Generally, refresh
tokens are used to extend the lifetime of a given authorization.

#### How to refresh

To refresh a token, you need an access token/refresh token pair coming from a body. For example

```bash
{
  "access_token": "17ebb971-f558-48f2-81b1-788ea927c509",
  "refresh_token": "c6f30bc4-9a04-4e66-a1a1-080fad703a9e",
  "expires_in": 3600,
  "token_type": "bearer"
}
```

You also need the `client_id` and `client_secret` used to generate the above access token/refresh token pair

To refresh, use this request:

```bash
POST https://pulsoid.net/oauth2/token
    --data-urlencode
    ?grant_type=refresh_token
    &refresh_token=<your refresh token>
    &client_id=<your client ID>
    &client_secret=<your client secret>
```

Parameters explained:

| Name   |      Type      |  Description |
|----------|:-------------:|------:|
| client_id |  string | Your client ID. |
| grant_type |  string | Should be `refresh_token`. |
| client_secret | string |        Your client secret.|
| refresh_token | string |    Refresh token issued to the client. |

Example:
```bash
POST https://pulsoid.net/oauth2/token
    --data-urlencode
    ?grant_type=refresh_token
    &refresh_token=c6f30bc4-9a04-4e66-a1a1-080fad703a9e
    &client_id=3d3fa070-8358-4984-ae32-94392185df63
    &client_secret=a8262283-f568-4ec3-be84-1c4758dc1a82
```

Here is a sample response on success. It contains the new access token, refresh token, and scopes associated with the new grant. Your application should then update its record of the refresh token to be the value provided in this response, as the refresh token may change between requests.

```bash
{
  "access_token": "79f4bbad-8894-4a04-9e4c-e36bfa0a9867",
  "refresh_token": "9ae58a4b-651a-41c1-a0fe-d3a50920da9b>",
  "expires_in": 3600,
  "token_type": "bearer"
}
```
After refresh the old refresh token and access token are invalid.
When a user disconnects an app, we delete all tokens for that user. Both refresh and access tokens for that user will return 401 Unauth.
We recommend performing a refresh when you receive a 401 Unauthorized.


### Manual Token issuing<a name="manual_token_issuing"></a>
To obtain a token go to https://pulsoid.net/ui/keys and click the button "Create new token". Note that token issuing is a BRO plan feature available under paid subscription or trial. OAuth2 auth protocol implementation is a work in progress. 
### Validate token<a name="validate"></a>

---------------
#### Request
| name        | value|           
| ------------- |:-------------:|
|url|`https://dev.pulsoid.net/api/v1/token/validate`|
|method|`GET`|

#### Response
| name        |type| description|           
| -------------|:-------------|:-------------:|
|client_id|string|Pulsoid oauth client id|
|expires_in|number|When token is going to expire in seconds|
|profile_id|string|Pulsoid profile id|
|scopes|array of strings|Token scopes|
#### Curl Request Example

```bash
curl --request GET \
  --url https://dev.pulsoid.net/api/v1/token/validate \
  --header 'Authorization: Bearer 8c4da3ce-7ed7-4a19-a1f1-058498661e45' \
  --header 'Content-Type: application/json' 
```
---------------
#### Response Example
```json
{
  "client_id": "4463b85f-79cd-445f-abee-70a0887f0a85",
  "expires_in": 625608812,
  "profile_id": "4fda99b9-6dc4-4ef4-9fe7-15f64908ad3f",
  "scopes": [
    "data:heart_rate:read",
    "data:heart_rate:write"
  ]
}
```
## List of supported scopes<a name="scopes"></a>
- `data:heart_rate:read`
- `profile:read`

## Errors<a name="errorsr"></a>
### Format
| name        |string| description|           
| -------------|:-------------|:-------------:|
|error_code|number|Error code|
|error_message|string|Human readable message|
### Example
```json
{
  "error_code": "7010",
  "error_message": "error_authorization_header_has_wrong_format"
}
```
### List
| error_code        |error_message|http code|           
| -------------|:-------------|:-------------:|
|7001|error_while_validating_token|500|
|7002|error_while_validating_token|500|
|7003|error_while_validating_token|500|
|7004|error_while_validating_token|500|
|7005|token_not_found|403|
|7006|token_expired|403|
|7007|premium_required|402|
|7009|error_authorization_header_is_not_present|403|
|7010|error_authorization_header_has_wrong_format|403|
|7011|error_invalid_scope|400|
|6001|error_while_validating_token|403|
|6002|error_retrieving_latest_heart_rate|403|
|6003|error_invalid_scope|400|

## Endpoints<a name="endpoints"></a>
### Read Heart Rate via Rest<a name="read_heart_rate_via_rest"></a> 
---------------
This method allows reading the latest heart rate of the user. Note that most heart rate monitors can measure changes in heart rate once per second, so it is reasonable to query this endpoint once per 500 ms to receive real-time heart rate data. 
#### Request
| name        | value|           
| ------------- |:-------------:|
|url|`https://dev.pulsoid.net/api/v1/data/heart_rate/latest`|
|method|`GET`|
|scope| `data:heart_rate:read`|

#### Response
| name        |type| description|           
| -------------|:-------------|:-------------:|
|measured_at|number|Unix timestamp in milliseconds|
|data|object|Holds metrics data|
|data.heart_rate|number|User's latest received heart rate|

#### Specific Errors
| http status code | reason|           
| -------------|:-------------:|
|412|User doesn't have any heart rate data|


#### Curl Request Example

```bash
curl --request GET \
  --url https://dev.pulsoid.net/api/v1/data/heart_rate/latest \
  --header 'Authorization: Bearer 8c4da3ce-7ed7-4a19-a1f1-058498661e45' \
  --header 'Content-Type: application/json' 
```
#### Response Example
```json
{
  "measured_at": 1625310655000,
  "data": {
    "heart_rate": 40
  }
}
```
---------------
### Read Heart Rate via Websocket<a name="read_heart_rate_via_websocket"></a> 
---------------
This method allows reading the heart rate of the user in real-time. Websocket connection can be interrupted at any point in time, make sure to have retry logic with backoff.
#### Request
| name        | value|           
| ------------- |:-------------:|
|url|`wss://dev.pulsoid.net/api/v1/data/real_time`|
|scope| `data:heart_rate:read`|
|query| `access_token`|

#### Websocket URL Request Example

```bash
wss://dev.pulsoid.net/api/v1/data/real_time?access_token=8c4da3ce-7ed7-4a19-a1f1-058498661e45
```
#### Websocket Message Example
```json
{
  "measured_at": 1625310655000,
  "data": {
    "heart_rate": 40
  }
}
```
---------------
### Read profile information<a name="read_profile"></a> 
---------------
This method allows reading the information of the user. It includes premium status, channel, and login(can be an email). 
#### Request
| name        | value|           
| ------------- |:-------------:|
|url|`https://dev.pulsoid.net/api/v1/profile`|
|method|`GET`|
|scope| `profile:read`|

#### Response
| name        |type| description|           
| -------------|:-------------|:-------------:|
|channel|string|User's channel e.g. twitch.tv/pulsoid|
|login|string|User's login e.g. pulsoid, support@pulsoid.net|

#### Curl Request Example

```bash
curl --request GET \
  --url https://pulsoid.net/api/v1/profile \
  --header 'Authorization: Bearer 8c4da3ce-7ed7-4a19-a1f1-058498661e45'
```
#### Response Example
```json
{
	"channel": "twitch.tv/pulsoid",
	"login": "support@pulsoid.net"
}
```
---------------
