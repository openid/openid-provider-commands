---
title: "OpenID Provider Commands 1.0 - draft 00"
abbrev: "openid-provider-commands"
workgroup : "OpenID Connect"
docname: openid-provider-commands-1_0
date: 2024-06-25

keyword: ["security", "openid", "lifecycle"]

ipr: none
cat: std
wg: connect

pi:
  toc: yes
  sortrefs: yes
  symrefs: yes
  private: yes


author:
      -
        ins: "D. Hardt"
        name: "Dick Hardt"
        org: "Hellō"
        email : "dick.hardt@gmail.com"

      -
        ins: "K. McGuinness"
        name: "Karl McGuinness"
        org: "Independent"
        email : "me@karlmcguinness.com"

normative:

  OpenID.Core:
    author:
    - ins: N. Sakimura
      name: Nat Sakimura
    - ins: J. Bradley
      name: John Bradley
    - ins: M.B. Jones
      name: Michael B. Jones
    - ins: B. de Medeiros
      name: Breno de Medeiros
    - ins: C. Mortimore
      name: Chuck Mortimore
    date: November 2014
    target: https://openid.net/specs/openid-connect-core-1_0.html#IDToken
    title: OpenID Connect Core 1.0 - ID Token



--- abstract

OpenID Connect defines a protocol for an end-user to use an OpenID Provider (OP) to log in to a Relying Party (RP) and assert Claims about the end-user using an ID Token. RPs will often use the identity Claims about the user to implicitly (or explicitly) establish an Account for the user at the RP

OpenID Provider Commands complements OpenID Connect by introducing a set of Commands for an OP to directly manage an end-user Account at an RP. These Commands enable an OP to activate, maintain, suspend, reactivate, archive, restore, delete, and unauthorize an end-user Account. Command Tokens build on the OpenID Connect ID Token schema and verification, simplifying adoption by RPs.

--- middle

# Introduction


OpenID Connect 1.0 is a widely adopted identity protocol that enables client applications, known as relying parties (RPs), to verify the identity of end-users based on authentication performed by a trusted service, the OpenID Provider (OP). OpenID Connect also provides mechanisms for securely obtaining identity attributes, or Claims, about the end-user, which helps RPs tailor experiences and manage access with confidence.

OpenID Connect not only allows an end-user to log in and authorize access to resources at an RP but may also be
used to implicitly or explicitly create an Account at the RP. However, Account creation is only the beginning of an Account's lifecycle. Throughout the lifecycle, various actions may be required to ensure data integrity, security, and regulatory compliance.

For example, many jurisdictions grant end-users the "right to be forgotten," enabling them to request the deletion of their Accounts and associated data. When such requests arise, OPs may need to notify RPs to fully delete the end-user's Account and remove all related data, respecting both regulatory obligations and end-user privacy preferences.

In scenarios where malicious activity is detected or suspected, OPs play a vital role in protecting end-users. They may need to instruct RPs to revoke authorization or delete Accounts created by malicious actors. This helps contain the impact of unauthorized actions and prevent further misuse of compromised Accounts.

In enterprise environments, where organizations centrally manage workforce access, OPs handle essential Account operations across various stages of the lifecycle. These operations include activating, maintaining, suspending, reactivating, archiving, restoring, and deleting Accounts to maintain security and compliance.

OpenID Provider Commands are a remote procedure call from the OP to the RP that enables OPs to manage the Account lifecycle, building upon the existing OP / RP relationship to cover the full spectrum of Account management requirements.


## Requirements Notation and Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{RFC2119}}.

The key word "PROHIBITED" is to be interpreted as "MUST NOT".

In the .txt version of this specification,
values are quoted to indicate that they are to be taken literally.
When using these values in protocol messages,
the quotes MUST NOT be used as part of the value.
In the HTML version of this specification,
values to be taken literally are indicated by
the use of *this fixed-width font*.

## Terminology

This specification defines the following terms:

- **Account**: The Claims about a user in the RPs identity register.

- **Command**: An instruction from an OP to an RP. It is a synchronous request and response.

- **Command Token**: A JSON Web Token (JWT) signed by the OP that contains Claims about the Command being issued.

- **Commands URI**: The URL at the RP where OPs post Command Tokens.

- **Tenant**: A logically isolated entity within an OP that represents a distinct organizational or administrative boundary. An OP may have a single Tenant, or multiple Tenants. The Tenant may contain Accounts managed by individuals, or may contain Accounts managed by an organization.

## Protocol Overview

This specification defines a Command Request containing a Command Token sent from the OP to the RP, and a Command Response returned from the RP to the OP.

```
+------+  Command request       +------+
|      |---- Command Token ---->|      |
|  OP  |                        |  RP  | 
|      |<-----------------------|      |
+------+      Command response  +------+
```

## Command Usage Overview

An OP will typically send a Metadata Command at the start of the relationship between a Tenant and an RP to share the OP's capabilities and metadata and to learn the Commands an RP supports, and other RP metadata. The OP will typically send the Metadata Command when there is a change in the OP capabilities or metadata, and periodically to learn of any RP changes. The OP may use the response from the Metadata Command to determine if the RP supports functionality required by the Tenant before issuing ID Tokens or Activate Commands to the RP.

If the RP supports any Account Commands, the OP will send supported Account Commands to synchronize the state of Accounts at the RP with the state at the Tenant. If the RP supports Account Commands, the RP should also support the Audit Tenant Command. The OP will typically send an Audit Tenant Command at the start of the Tenant and RP relationship, and then periodically, to learn the state of the Tenant's Accounts at the RP and correct any drift between the Account state at the Tenant and the RP.

If the RP supports the Unauthorize Command, the OP will send the Unauthorize Command if the OP suspects an Account has been taken over by a malicious actor.

A Tenant with Accounts managed by individuals will typically only support the Metadata, Unauthorize, and Delete Commands.

# Command Request

The OP uses an HTTP POST to the registered Commands URI
to send Account Commands to the RP. The POST body uses the
`application/x-www-form-urlencoded` encoding
and must include a `command_token` parameter
containing a Command Token from the OP for the RP.

The POST body MAY contain other values in addition to
`command_token`.
Values that are not understood by the implementation MUST be ignored.

The following is a non-normative example of such a Command request
(with most of the Command Token contents omitted for brevity):

```
POST /commands HTTP/1.1
Host: rp.example.net
Content-Type: application/x-www-form-urlencoded

command_token=eyJhbGci ... .eyJpc3Mi ... .T3BlbklE ...
```

# Command Response

If the Command succeeded, the RP MUST respond with HTTP 200 OK.
However, note that some Web frameworks will substitute an HTTP 204 No Content response
for an HTTP 200 OK when the HTTP body is empty.
Therefore, OPs should be prepared to also process an HTTP 204 No Content response as a successful response.


The RP's response MUST include the Cache-Control HTTP response header field with a no-store value, keeping the response from being cached to prevent cached responses from interfering with future Command requests. An example of this is:

```
Cache-Control: no-store
```

If there is a response body, it MUST be JSON and use the `application/json` media type.

If the request is not valid, the RP MUST return an `error` parameter, and may include a `error_description` parameter. 
Note that the information conveyed in an error response is intended to help debug deployments;
it is not intended that implementations use different `error` values
to trigger different runtime behaviors.


## Invalid Request Error

If there was a problem with the syntax of the Command request, or the Command Token was invalid, the RP MUST return an HTTP 400 Bad Request and include the `error` parameter with a value of `invalid_request`. 

## Unrecognized Provider Error

If the RP does not recognize the OP identified by the `iss` value in the Command token, the RP MUST return an HTTP 401 Unauthorized response and include the `error` parameter with the value of `unrecognized_provider`.

## Unsupported Command Error

If the RP does not support the Command requested, the RP MUST return an HTTP 400 Bad Request and include the `error` parameter with the value of `unsupported_command`.

Note that the RP may support Commands for some OPs, and not others, and for some Tenants, and not others.

## Server Error

If the RP is unable to process a valid request, the RP MUST respond with a 5xx Server Error status code as defined in RFC 9110 section 15.6.

# Command Token

OPs send a JWT similar to an ID Token to RPs called a Command Token
to issue Commands. ID Tokens are defined in Section 2 of {{OpenID.Core}}.

The following Claims are used within the Command Token:

- **iss**  
  REQUIRED.  
  Issuer Identifier, as specified in Section 2 of {{OpenID.Core}}.

- **sub**  
  REQUIRED for Account Commands.    
  PROHIBITED for Tenant Commands. 

  Subject Identifier, as specified in Section 2 of {{OpenID.Core}}. 
  
  The combination of the `iss` and `sub` Claim uniquely identifies the Account for the Command.

- **aud**  
  REQUIRED.  
  Audience(s), as specified in Section 2 of {{OpenID.Core}}.

- **iat**  
  REQUIRED.  
  Issued at time, as specified in Section 2 of {{OpenID.Core}}.

- **exp**  
  REQUIRED.  
  Expiration time, as specified in Section 2 of {{OpenID.Core}}.

- **jti**  
  REQUIRED.  
  Unique identifier for the token, as specified in Section 9 of {{OpenID.Core}}.

- **command**  
  REQUIRED.  
  A JSON string. The Command for the RP to execute. See [Account Commands](#account-commands) and [Tenant Commands](#tenant-commands) for standard values defined in this document. Other specifications may define additional values as described in [Extensibility](#defining-new-commands).

- **tenant**  
  REQUIRED.  
  A JSON string. The Tenant identifier. MAY have the value `personal`, `organization` or a stable OP unique value for multi-tenant OPs. The `personal` value is reserved for when Accounts are managed by individuals. The `organization` value is reserved for Accounts are managed by an organization.

  The combination of `iss` and `tenant` uniquely identifies a Tenant. 

Commands may define additional REQUIRED or OPTIONAL Claims.
An OP MAY include additional Claims. Any Claims that are not understood by the RP MUST be ignored.
  
The following Claim MUST NOT be used within the Command Token:

- **nonce**  
  PROHIBITED.  
  A `nonce` Claim MUST NOT be present.
  Its use is prohibited to prevent misuse of the Command Token. See [Cross-JWT Confusion](#cross-jwt-confusion) for details.


A Command Token MUST be signed.
The same keys an OP uses to sign ID Tokens are used to sign Command Tokens, allowing key discovery using the same mechanism used for ID Tokens.


Command Tokens MUST be explicitly typed.
This is accomplished by including a `typ` (type) Header Parameter
with a value of `command+jwt` in the Command Token.
See [Security Considerations](#security-considerations) for a discussion of the security and interoperability considerations
of using explicit typing.

>
> Add example of JWT header to show "typ":"command+jwt"?
>

A non-normative example JWT Claims Set for the Command Token for an Activate Command follows:

```json
{
  "iss": "https://op.example.org",
  "sub": "248289761001",
  "aud": "s6BhdRkqt3",
  "iat": 1734003000,
  "exp": 1734003060,
  "jti": "bWJq",
  "command": "activate",
  "given_name": "Jane",
  "family_name": "Smith",
  "email": "jane.smith@example.org",
  "email_verified": true,
  "tenant": "ff6e7c9",
  "groups": [
    "b0f4861d",
    "88799417"
  ]
}
```

A non-normative example JWT Claims Set for the Command Token for an Unauthorize Command follows:

```json
{
  "iss": "https://op.example.org",
  "sub": "248289761001",
  "aud": "s6BhdRkqt3",
  "iat": 1734004000,
  "exp": 1734004060,
  "jti": "bWJr",
  "command": "unauthorize",
}
```



# Account Commands

Account Commands that operate on an Account. Support for any Account Command is OPTIONAL. Account Commands are executed on an RP Account identified by the `iss` and `sub` Claims in a Command Token. Account Commands include Lifecycle Commands and the Unauthorize Command.


## Account Lifecycle States

Lifecycle Commands are defined for each transition defined in ISO 24760-1 (2019-05 edition), Section 7.2 for Accounts in an RP's identity register as defined in {{ISO}} 3.4.5.

Lifecycle Commands transition the Account between the following states:

- **unknown**

- **active**

- **suspended**

- **archived**

Following are the potential state transitions:

```
                      +--------------------------------------- reactivate ---+                   
                      |  +--- maintain --+                                   |
                      |  |               |                                   |
+---------+           |  |   +--------+  |                    +-----------+  |
|         |           |  +-> |        | -+                    |           | -+ 
| unknown |           + ---> | active | -------- suspend ---> | suspended | --------+
|         | --- activate --> |        | ----+                 |           | -+      |
+---------+              +-> |        | -+  |                 +-----------+  |      |
  ^  ^  ^                |   +--------+  |  |                                |      |
  |  |  |                |               |  |            +------ archive† ---+      | 
  |  |  |                |               |  |            |                          |
  |  |  |                |               |  |            |     +-----------+        |
  |  |  |                |               |  |            +---> |           | ----+  | 
  |  |  |                |               |  +--- archive ----> | archived  | -+  |  |            
  |  |  |                |               |                     |           |  |  |  |
  |  |  |                |               |                     +-----------+  |  |  |
  |  |  |                |               |                                    |  |  |
  |  |  |                +---------------|----------------------- restore ----+  |  | 
  |  |  |                                |                                       |  |   
  |  |  +--------------------- delete ---+                                       |  |
  |  +------------------------ delete -------------------------------------------+  | 
  +--------------------------- delete ----------------------------------------------+
```                                             

†The transition from **suspended** to **archived** is an extension to the ISO standard.

## Success Response

When an RP successful processes an Account Command, the RP returns an HTTP 200 Ok and a JSON object containing `account_state` set to the state of the Account after processing. 

Following is a non-normative response to a successful Activate Command:

```json
{
  "account_state": "active"
}
```

## Incompatible State Response

If the Account is in an incompatible state in the identity register for the Account Command, the RP returns an HTTP 409 Conflict and a JSON object containing:

- **account_state** where the value is the current state of the Account in the identity register
- **error** set to the value `incompatible_state`

Following is a non-normative response to a unsuccessful Restore Command where the Account was in the **suspended** state:

```json
{
  "account_state": "suspended",
  "error": "incompatible_state"
}
```

Note that if an Activate Command is sent for an Account that exists, or one of the other Commands are sent for an Account that does not exist, 
the Account is incompatible state. 

Following is a non-normative response to an unsuccessful Activate Command for an existing Account in the **active** state:

```json
{
  "account_state": "active",
  "error": "incompatible_state"
}
```

Following is a non-normative response to an unsuccessful Maintain Command for a non-existing Account:

```json
{
  "account_state": "unknown",
  "error": "incompatible_state"
}
```


## Activate Command
Identified by the `activate` value in the `command` Claim in a Command Token.

The RP MUST create an Account with the included Claims in the identity register. The Account MUST be in the **unknown** state. The Account is in the **active** state after successful processing.

## Maintain Command
Identified by the `maintain` value in the `command` Claim in a Command Token.

The RP MUST update an existing Account in the identity register with the included Claims. The Account MUST be in the **active** state. The Account remains in the **active** state after successful processing.

## Suspend Command
Identified by the `suspend` value in the `command` Claim in a Command Token.

The RP MUST perform the [Unauthorize Functionality](#unauthorize-functionality) on the Account and mark the Account as being temporarily unavailable in the identity register. The Account MUST be in the **active** state. The Account is in the **suspended** state after successful processing.


## Reactivate Command
Identified by the `reactivate` value in the `command` Claim in a Command Token.

The RP MUST mark a suspended Account as being active in the identity register. The Account MUST be in the **suspended** state. The Account is in the **active** state after successful processing. The RP SHOULD support the Reactivate Command if it supports the Suspend Command.

## Archive Command
Identified by the `archive` value in the `command` Claim in a Command Token.

The RP MUST perform the [Unauthorize Functionality](#unauthorize-functionality) on the Account and remove the Account from the identity register. The Account MUST be in either the **active** or **suspended** state. The Account is in the **archived** state after successful processing.



## Restore Command
Identified by the `restore` value in the `command` Claim in a Command Token.

The RP MUST restore an archived Account to the identity register and mark it as being active. The Account MUST be in the **archived** state. The Account is in the **active** state after successful processing. The RP SHOULD support the Restore Command if it suppSs the chA Comma.

## Delete Command
Identified by the `delete` value in the `command` Claim in a Command Token.

The RP MUST perform the [Unauthorize Functionality](#unauthorize-functionality) on the Account, and delete all data associated with an Account. The Account can be in any state except **unknown**. The Account is in the **unknown** state after successful processing.

## Unauthorize Command

Identified by the `unauthorize` value in the `command` Claim in a Command Token.

The RP MUST perform the [Unauthorize Functionality](#unauthorize-functionality) on the Account.
The OP sends this Command when it wants the RP to undo the last OpenID Connect login. The OP may send this when it suspects a previous OpenID Connect ID Token issued by the OP was granted to a malicious actor, or if the user's device was compromised. 


The Account MUST be in the **active** state, and remains in the **active** state after executing the Command. If the Account is in any other state, the RP MUST return an [Incompatible State Response](#incompatible-state-response).

The functionality of the Unauthorize Command is also performed by Suspend, Archive, and Delete Commands to ensure an Account in the **suspended**, **archived**, and **unknown** state no longer has access to resources.

## Unauthorize Functionality

The RP MUST revoke all active sessions and MUST reverse all authorization that may have been granted to applications, including `offline_access`, for Account resources identified by the `sub`.








# Tenant Commands

Tenant Commands are executed in the context of a Tenant. The RP MUST support the Metadata Command. Support for other Tenant Commands is optional. Command Tokens for Tenant Commands MUST contain the `tenant` Claim, and are PROHIBITED from containing the `sub` Claim.

## Metadata Command

Identified by the `metadata` value in the `command` Claim in a Command Token.

The OP sends this Command to exchange metadata with the RP. The OP sends its metadata in the Command Request, and the RP sends its metadata in the Command Response.

The OP sends this Command to inform the RP of any domains or groups for a Tenant, and any other Tenant metadata the OP wants to share with the RP.

A Metadata Command replaces any previous metadata provided by the OP to the RP in the Command Request, and any metadata provided by the RP to the OP in the Command Response. 


The Claims set in a Metadata Command Token MUST include the following claim:

- **metadata**
  REQUIRED.
  A JSON object that MAY include the following Claims:

  - **domains**
    OPTIONAL.
    A JSON array of one or more domain names the OP has verified the Tenant controls.

  - **groups**
    OPTIONAL.
    A JSON array of objects that MUST contain:
    - **id**
      REQUIRED.
      The OP unique value for the group.

    - **display**
      REQUIRED.
      The Tenant unique human readable name for the group.

The OP sends the `domains` array for the RP to link any data the RP has to the OP Tenant.

The OP sends the `groups` array to provide the display value for each identifier that the OP MAY include in a `groups` Claim in an ID Token or a Command Token for the Activate and Maintain Commands. This allows an admin at the RP to map centrally managed `groups` from an OP to roles or entitlements at an RP.

The OP MAY include any other metadata in the `metadata` Claim, including metadata defined in OAuth Authorization Server Metadata. **add reference**

Following is a non-normative example of a Claim set in a Command Token for the Metadata Command:

```json
{
  "iss": "https://op.example.org",
  "aud": "s6BhdRkqt3",
  "iat": 1734006000,
  "exp": 1734006060,
  "jti": "bWJt",
  "command": "metadata",
  "tenant": "ff6e7c96",
  "metadata": {
    "groups": [
      {
        "id": "b0f4861d",
        "display": "Administrators"
      },
      {
        "id": "88799417",
        "display": "Finance"
      }
    ],
    "domains": ["example.com"],
    "claims_supported": [
      "sub",
      "email",
      "email_verified",
      "name",
      "given_name",
      "family_name",
      "groups"
    ]
  }
}
```


## Metadata Response

If the Command Token is valid, the RP responds with an `application/json` media type that MUST include:

- **context**: a JSON object.
  - **iss**
  REQUIRED. the `iss` value from the Command Token.
  - **tenant**
  REQUIRED. the `tenant` value from the Command Token.
  
- **commands_supported**: a JSON array of Commands the RP supports. The `metadata` value MUST be included.
- **commands_uri**: the RP's Commands URI. This is the URL the Command Token was sent to.
- **describe_ttl**: the time in seconds the Command Response to the Metadata Command is valid for.
- **client_id**: the `client_id` for the RP.


The response MAY also include any OAuth Dynamic Client Registration Metadata *TBD [IANA reference](https://www.iana.org/assignments/oauth-parameters/oauth-parameters.xhtml#client-metadata)*

The RP MAY provide a different response for different **iss** Claim values, and for different `tenant` Claim values.

Following is a non-normative example of Command Response for a Metadata Command:

>
> NOTE: we want to include the `iss` and `tenant` from the Metadata Command in the Command Response so the OP knows the response from the RP
> is really for the correct `iss` and `tenant`. We don't want the OP `iss` at the top level as that would
> cause confusion if the Command Response is signed in the future since the `iss` then would be the RP
>

```json
{
  "context": {
    "iss": "https://op.example.org",
    "tenant":"73849284748493"
  },
  "commands_uri": "https://rp.example.net/commands",
  "commands_supported":[
    "describe",
    "unauthorize",
    "suspend",
    "reactivate",
    "delete",
    "audit"
  ],
  "commands_ttl": 86400,
  "claims_supported": [
    "sub",
    "email",
    "email_verified",
    "name",
    "groups"
  ],
  "client_id": "s6BhdRkqt3",
  "client_name": "Example RP",
  "logo_uri": "https://rp.example.net/logo.png",
  "policy_uri": "https://rp.example.net/privacy-policy.html",
  "tos_uri": "https://rp.example.net/terms-of-service.html",
  "jwks_uri": "https://rp.example.net/jwks",
  "initiate_login_uri": "https://rp.example.net/initiate-login",
  "redirect_uris": [
    "https://rp.example.net/response"
  ]
}
```

## Streaming Request

All Tenant Commands besides the Metadata Command use [Server-Side Events](https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events) (SSE) for the RP to transfer the Command Response. When performing a Streaming Request, the OP MUST include the following HTTP headers when sending the Command Request:

- `Accept` with the value of `text/event-stream` to indicate support for Server-Sent Events.
- `Cache-Control` with the value of `no-cache` to signal to intermediaries to not cache the response.
- `Connection` with the value of `keep-alive` to keep the connection open.


It is RECOMMENDED the OP accept compression of the response by sending the `Accept-Encoding` HTTP header with the value of `gzip`, or `gzip, br`. 

The following is a non-normative example of a Streaming Request:
(with most of the Command Token contents omitted for brevity):

```
POST /commands HTTP/1.1
Host: rp.example.net
Content-Type: application/x-www-form-urlencoded
Accept: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
Accept-Encoding: gzip, br

command_token=eyJhbGci ... .eyJpc3Mi ... .T3BlbklE ...
```


## Streaming Response

The RP sends a Streaming Response to a Streaming Request. In a Streaming Response, the RP uses SSE to stream the Command Response as a sequence of events. If the RP receives a valid Command, it MUST sent the `HTTP/1.1 200 OK` response, followed by the following headers:

- `Content-Type` with the `text/event-stream` value
- `Cache-Control` with the `no-cache` value
- `Connection` with the `keep-alive` value

If the OP sent a `Content-Encoding` header in the request with a compression the RP understands, the RP MAY include a `Content-Encoding` header with one of the OP provided values.

Per SSE, the body of the response is a series of events. In addition to the required field name `data`, each event MUST include the `id` field with a unique value for each event, and the `event` field with a value of either `account-state`, or `command-complete`. The RP sends an `account-state` event for each Account at the RP for the `iss`, and `org` if sent, in the Audit Tenant Command. When all `account-state` events have been sent, the RP sends an `command-complete` event.

The `data` parameter of the `account-state` event MUST contain the following:

- `sub` representing the Account
- `account_state` representing the current state for the Account from the states supported by the RP.

The `data` parameter MAY include other Claims as defined by the Tenant Command.

The `data` parameter of the `command-complete` event MUST include the `total_accounts` property with a value for the total number of `account-state` events the RP has sent.

If there are no Accounts for the Tenant at the RP, the RP responds with only the `command-complete` event with `total-accounts` having a value of `0`.


The following is a non-normative example of a Streaming Response for an Audit Tenant Command:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
Content-Encoding: gzip

id: 1
event: account-state
data: {
  "sub": "248289761001",
  "email": "janes.smith@example.com",
  "given_name": "Jane",
  "family_name": "Smith",  
  "groups": [
    "b0f4861d-f3d6-4f76-be2f-e467daddc6f6",
    "88799417-c72f-48fc-9e63-f012d8822ad1"
  ],
  "account_state": "active"
}

id: 2
event: account-state
data: {
  "sub": "98765412345",
  "email": "john.doe@example.com",
  "given_name": "John",
  "family_name": "Doe",
  "groups": [
    "88799417-c72f-48fc-9e63-f012d8822ad1"
  ],
  "account_state": "suspended"
}

id: 3
event: command-complete
data: {
  "total_accounts": 2
}
```


If the connection is lost during a Streaming Response, The OP SHOULD generate a new Command Token and send a Streaming Request include the HTTP header `Last-Event-Id` with the last event `id` property received per SSE. 

Following is a non-normative example of a Streaming Request sent after a connection was lost:

```
POST /commands HTTP/1.1
Host: rp.example.net
Content-Type: application/x-www-form-urlencoded
Accept: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
Accept-Encoding: gzip, br
Last-Event-Id: 3

command_token=eyJhbGci ... .eyJpc3Mi ... .T3BlbklE ...
```

## Audit Tenant Command

Sent in a Streaming Request and identified by the `audit_tenant` value in the `command` Claim in a Command Token.

The OP sends the Audit Tenant Command to learn the state of Accounts for a Tenant at an RP. 

The following is a non-normative example of the Claims Set in the Command Token of an Audit Tenant Command:

```json
{
  "iss": "https://op.example.org",
  "aud": "s6BhdRkqt3",
  "iat": 1734003000,
  "exp": 1734003060,
  "jti": "bWJz",
  "command": "audit_tenant",
  "tenant": "ff6e7c9"
}
```


## Audit Tenant Response

The RP sends a Streaming Response if it received a valid Suspend Tenant Command.

The RP MUST include any Claims for an Account that the RP has retained that were provided by the OP in the event `data` parameter JSON string. If the Claim values have been modified at the RP, the modified values should be returned. 

## Suspend Tenant Command

Sent in a Streaming Request and identified by the `suspend_tenant` value in the `command` Claim in a Command Token.
Upon receiving this Command, the RP MUST suspend all Accounts in the active state for the Tenant identified by the `tenant` Claim in the Command Token. 

The RP sends a Streaming Response if it received a valid Suspend Tenant Command.

## Archive Tenant Command

Sent in a Streaming Request and identified by the `archive_tenant` value in the `command` Claim in a Command Token.
Upon receiving this Command, the RP MUST suspend all Accounts in the active and suspended state for the Tenant identified by the `tenant` Claim in the Command Token.

The RP sends a Streaming Response if it received a valid Archive Tenant Command.


## Delete Tenant Command

Sent in a Streaming Request and identified by the `delete_tenant` value in the `command` Claim in a Command Token.
Upon receiving this Command, the RP MUST delete all Accounts for the Tenant identified by the `tenant` Claim in the Command Token.

The RP sends a Streaming Response if it received a valid Delete Tenant Command. If the Delete Tenant Command was successful, the RP will send only a `command_completed` event with a `data` parameter containing the JSON string `{"total_accounts":0}`.

## Unauthorize Tenant Command

Sent in a Streaming Request and identified by the `unauthorize_tenant` value in the `command` Claim in a Command Token.
Upon receiving this Command, the RP MUST perform the [Unauthorize Functionality](#unauthorize-functionality) on all Accounts in the **active** state for the Tenant identified by the `tenant` Claim in the Command Token.

The RP sends a Streaming Response if it received a valid Unauthorize Tenant Command.

# Extensibility


## Defining New Commands

New Commands can be defined in one of two ways: registered in the OpenID Provider Commands 
registry, or by using a unique absolute URI as its name.

Types utilizing a URI name SHOULD be limited to vendor-specific
implementations that are not commonly applicable, and are specific to
the implementation details of the resource server where they are
used.

## Defining New Tenant Metadata

*To be completed.*


## Defining New Relying Party Metadata

*To be completed.*


# OpenID Provider Command Support

Relying Parties supporting OpenID Provider Commands register a Commands URI with the OP as part of their client registration.

The Commands URI MUST be an absolute URI as defined by
Section 4.3 of {{RFC3986}}.
The Commands URI MAY include an
`application/x-www-form-urlencoded` formatted
query component, per Section 3.4 of {{RFC3986}}.
The Commands URI MUST NOT include a fragment component.

If the RP supports
[OpenID Connect Dynamic Client Registration 1.0](#OpenID.Registration),
it uses this metadata value to register the OpenID Provider Commands URI:

- **commands_uri**  
  OPTIONAL.  
  RP URL that will receive OpenID Provider Commands from the OP.
  This URL MUST use the `https` scheme
  and MAY contain path, and query parameter components.

# Implementation Considerations

This specification defines features used by both Relying Parties and
OpenID Providers that choose to implement OpenID Provider Commands.
All of these Relying Parties and OpenID Providers
MUST implement the features that are listed
in this specification as being "REQUIRED" or are described with a "MUST".
No other implementation considerations for implementations of
OpenID Provider Commands are defined by this specification.

# Security Considerations

The signed Command Token is required in the Command Request to prevent
denial of service attacks by enabling the RP to verify that
the Command Request is coming from a legitimate party.

OPs are encouraged to use short expiration times in Command Tokens,
preferably at most two minutes in the future,
to prevent captured Command Tokens from being replayed.


## Cross-JWT Confusion

As described in Section 2.8 of {{RFC8725}},
attackers may attempt to use a JWT issued for one purpose in a context that it was not intended for.
The mitigations described for these attacks can be applied to Command Tokens.

One way that an attacker might attempt to repurpose a Command Token
is to try to use it as an ID Token.
As described in [Command Token](#command-token),
inclusion of a `nonce` Claim in a Command Token
is prohibited to prevent its misuse as an ID Token.

Another way to prevent cross-JWT confusion is to use explicit typing,
as described in Section 3.11 of {{!RFC8725}} and as required in [#command-token]. 


# Privacy Considerations

*To be completed.*

# IANA Considerations

*Not all entries have been added.*

## JSON Web Token Claims

This specification registers the following JSON web token claim definitions
in the IANA "JSON Web Token Claims" registry
[IANA JSON Web Token Claims](#IANA.JSON.Web.Token.Claims)
established by [RFC7519](#RFC7519).

### Registry Contents

- **Claim Name:** `command`

**Claim Description:**
  An instruction from the `iss` to the `aud` of a token.

**Change Controller:** OpenID Foundation

**Specification Document(s):** This document

- **Claim Name:** `tenant`

**Claim Description:**
  A logically isolated entity within the party identified by the `iss` of the token.
  
**Change Controller:** OpenID Foundation

**Specification Document(s):** This document

- **Claim Name:** `metadata`

**Claim Description:**
  Metadata about the party identified by the `iss` of the token.
  
**Change Controller:** OpenID Foundation

**Specification Document(s):** This document


## OAuth Dynamic Client Registration Metadata Registration

This specification registers the following client metadata definitions
in the IANA "OAuth Dynamic Client Registration Metadata" registry
[IANA OAuth Parameters](#IANA.OAuth.Parameters)
established by [RFC7591](#RFC7591).

### Registry Contents

- **Client Metadata Name:** `commands_uri`
  
**Client Metadata Description:**
  RP URL that will receive OpenID Provider Commands from the OP

**Change Controller:** OpenID Foundation

**Specification Document(s):** This document


## Media Type Registration

This specification registers the `application/command+jwt` media type as per {{!RFC6838}}.

*To be completed.*


# References

## Normative References

*To be completed.*

## Informative References

*To be completed.*

# Acknowledgements

The authors would like to thank early feedback provided by Tim Cappalli, Pam Dingle, George Fletcher, Michael Jones, Aaron Parecki, Dean Saxe, and Rifaat Shekh-Yusef.

*To be updated.*

# Notices


Copyright (c) 2025 The OpenID Foundation.

The OpenID Foundation (OIDF) grants to any Contributor, developer, implementer, or other interested party a non-exclusive, royalty free, worldwide copyright license to reproduce, prepare derivative works from, distribute, perform and display, this Implementers Draft or Final Specification solely for the purposes of (i) developing specifications, and (ii) implementing Implementers Drafts and Final Specifications based on such documents, provided that attribution be made to the OIDF as the source of the material, but that such attribution does not indicate an endorsement by the OIDF.

The technology described in this specification was made available from contributions from various sources, including members of the OpenID Foundation and others. Although the OpenID Foundation has taken steps to help ensure that the technology is available for distribution, it takes no position regarding the validity or scope of any intellectual property or other rights that might be claimed to pertain to the implementation or use of the technology described in this specification or the extent to which any license under such rights might or might not be available; neither does it represent that it has made any independent effort to identify any such rights. The OpenID Foundation and the contributors to this specification make no (and hereby expressly disclaim any) warranties (express, implied, or otherwise), including implied warranties of merchantability, non-infringement, fitness for a particular purpose, or title, related to this specification, and the entire risk as to implementing this specification is assumed by the implementer. The OpenID Intellectual Property Rights policy requires contributors to offer a patent promise not to assert certain patent claims against other contributors and against implementers. The OpenID Foundation invites any interested party to bring to its attention any copyrights, patents, patent applications, or other proprietary rights that may cover technology that may be required to practice this specification.


# Document History

   [[ To be removed from the final specification ]]

   -00

   initial draft