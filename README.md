# Pulsoid API


## Table of contents
1. [General integration information](#general_information)
2. [Authentication](#authentication)
	1. [OAuth2](#oauth2)
	2. [Manual Token issuing](#manual_token_issuing)
	3. [Validate token](#validate)
3. [Scopes](#scopes)
4. [Errors](#errorsr)
5. [Endpoints](#endpoints)
	1. [Read Heart Rate via Rest](#read_heart_rate_via_rest)
	2. [Read Heart Rate via WebSocket](#read_heart_rate_via_websocket)
	3. [Write Heart Rate via Rest](#write_heart_rate_via_rest)(not implemented)

## General integration information<a name="general_information"></a>
Welcome to the Pulsoid integration guide. Root Pulsoid API domain is `https://dev.pulsoid.net`. All communication is done in JSON. All endpoints have a soft rate limit of 20 requests per second. 
## Authentication<a name="authentication"></a>
### OAuth2 <a name="oauth2"></a>
Contact support@pulsoid.net for further information.
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
- `data:heart_rate:write`(not implemented)
- `profile:read`(not implemented)

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
|timestamp|number|Unix timestamp in milliseconds|
|data|object|Holds metrics data|
|data.heart_rate|number|User's latest received heart rate|
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
  "timestamp": 1625310655000,
  "data": {
    "heart_rate": 40
  }
}
```
---------------
### Read Heart Rate via Rest<a name="read_heart_rate_via_websocket"></a> 
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
  "timestamp": 1625310655000,
  "data": {
    "heart_rate": 40
  }
}
```
---------------
