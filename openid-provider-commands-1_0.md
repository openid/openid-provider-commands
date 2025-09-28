%%%
title = "OpenID Provider Commands 1.0 - draft 02"
abbrev = "openid-provider-commands"
ipr = "none"
workgroup = "OpenID Connect"
keyword = ["security", "openid", "lifecycle"]

[seriesInfo]
name = "Internet-Draft"
value = "openid-connect-commands-1_0"
status = "standard"

[[author]]
initials="D."
surname="Hardt"
fullname="Dick Hardt"
organization="Hellō"
    [author.address]
    email = "dick.hardt@gmail.com"

[[author]]  
initials="K."
surname="McGuinness"
fullname="Karl McGuinness"
organization="Independent"
    [author.address]
    email = "me@karlmcguinness.com"

%%%

.# Abstract

OpenID Connect defines a protocol for an end-user to use an OpenID Provider (OP) to log in to a Relying Party (RP) and assert Claims about the end-user using an ID Token. RPs will often use the identity Claims about the user to implicitly (or explicitly) establish an Account for the user at the RP

OpenID Provider Commands complements OpenID Connect by introducing a set of Commands for an OP to directly manage an end-user Account at an RP. These Commands enable an OP to activate, maintain, suspend, reactivate, archive, restore, delete, audit, invalidate an end-user Account, and migrate authentication of an existing account. Command Tokens build on the OpenID Connect ID Token schema and verification, simplifying adoption by RPs.


{mainmatter}

# Introduction


OpenID Connect 1.0 is a widely adopted identity protocol that enables client applications, known as relying parties (RPs), to verify the identity of end-users based on authentication performed by a trusted service, the OpenID Provider (OP). OpenID Connect also provides mechanisms for securely obtaining identity attributes, or Claims, about the end-user, which helps RPs tailor experiences and manage access with confidence.

OpenID Connect not only allows an end-user to log in and authorize access to resources at an RP but may also be
used to implicitly or explicitly create an Account at the RP. However, Account creation is only the beginning of an Account's lifecycle. Throughout the lifecycle, various actions may be required to ensure data integrity, security, and regulatory compliance.

For example, many jurisdictions grant end-users the "right to be forgotten", enabling them to request the deletion of their Accounts and associated data. When such requests arise, OPs may need to notify RPs to fully delete the end-user's Account and remove all related data, respecting both regulatory obligations and end-user privacy preferences.

In scenarios where malicious activity is detected or suspected, OPs play a vital role in protecting end-users. They may need to instruct RPs to revoke authorization or delete Accounts created by malicious actors. This helps contain the impact of unauthorized actions and prevent further misuse of compromised Accounts.

In enterprise environments, where organizations centrally manage workforce access, OPs need to establish control over existing employee accounts at RPs. First, the OP must establish a bidirectional mapping between its account identifiers and the RP's internal identifiers through account resolution, enabling both parties to refer to the same specific account. Once this identifier resolution is complete, enterprises can migrate authentication responsibility for accounts their employees may have previously created using RP-managed credentials or other authentication methods. With authentication control and identifier mapping established, the OP can then handle essential Account operations across various stages of the lifecycle, including activating, maintaining, suspending, reactivating, archiving, restoring, and deleting Accounts to maintain security and compliance.
  
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

- **Synchronous Command**: A Command where the response from the RP is provided synchronously to the OP with an HTTP 200 Success response.

- **Asynchronous Command**: A Command where the response from the RP is provided asynchronously to the OP with an initial HTTP 202 Accepted response. The RP sends the final response of an Asynchronous Command to the OP **notification_endpoint**. Command identifiers for Asynchronous Commands end with the string `_async`. 

- **Command Token**: A JSON Web Token (JWT) signed by the OP that contains Claims about the Command being issued by the OP.

- **Command Endpoint**: The URL at the RP where OPs post Command Tokens.

- **Notification**: A message sent from the RP to an OP to notify the OP of a change in RP state or to send async command responses.

- **Notification Token**: An JSON Web Token (JWT) signed by the RP that contains Claims about the Notification from the RP.


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

The OP may provide a notification endpoint for the RP to send Notification Tokens to notify the OP that the RP's state has changed or to send the results of an asynchronous command. 

```
+------+          Notification  +------+
|      |<- Notification Token --|      |
|  OP  |                        |  RP  | 
|      |----------------------->|      |
+------+  204 or error response +------+
```


## Command Use Cases

OpenID Provider Commands support several distinct use cases, each with different command sequences and requirements. Understanding these use cases helps implementers determine which commands to support and how to sequence them effectively.

### Personal Applications

Personal applications serve individual users who manage their own accounts. In this scenario, accounts are created and managed by end-users rather than organizations.

**Supported Commands**: Metadata, Invalidate, Delete

**Typical Command Sequence**:
1. **Metadata Command** - Establish capabilities between OP and RP
2. **Invalidate Command** - Invalidate all sessions and tokens if account compromise is suspected  
3. **Delete Command** - Remove account and data upon user request (e.g., "right to be forgotten")

### Enterprise Application Migration

When organizations implement new identity providers or undergo mergers/acquisitions, they need to migrate authentication responsibility for existing accounts. This commonly occurs when individual users initially create accounts with RP-managed credentials, and organizations later need to centralize authentication under their corporate identity provider.

**Migration from RP-managed to OP-managed Authentication**:
1. **Metadata Command** - Establish OP capabilities with target RP
2. **Audit Tenant Command** - Identify accounts requiring migration and establish account resolution
3. **Migrate Command** - Assume authentication responsibility for existing RP-managed accounts
4. **Account Lifecycle Commands** - Manage migrated accounts through their lifecycle

**Account Resolution in Migration**: Account Resolution is the process by which an OP and RP establish a bidirectional mapping between their respective account identifiers. When an OP receives an `aud_sub` value from an RP in response to commands such as Audit, the OP learns the RP's internal identifier for that account. This enables the OP to include the `aud_sub` claim in subsequent Command Tokens, allowing the RP to execute commands using its own identifiers rather than maintaining foreign key relationships based on the OP's `iss` and `sub` values. This simplifies RP implementation by eliminating the need to store and manage external identifiers, while still maintaining the ability to correlate accounts across systems.

**Authentication Migration Process**: Authentication Migration enables an OP to become the authentication provider for existing accounts through the Migrate Command. This process is essential during organizational migrations, acquisitions, or when implementing centralized identity governance policies. Account Resolution facilitates an OP becoming the authentication provider for an account with the Migrate Command.

### Enterprise Application Integration

Enterprise applications that are integrated with organizational identity providers need ongoing account management as employees join, change roles, or leave the organization.

**Supported Commands**: Metadata, Audit Tenant, Account Lifecycle Commands (Activate, Maintain, Suspend, Reactivate, Archive, Restore, Delete), Invalidate, Migrate

**Typical Command Sequence**:
1. **Metadata Command** - Exchange capabilities and establish relationship
2. **Audit Tenant Command** - Discover existing accounts and establish account resolution
3. **Account Lifecycle Commands** - Synchronize account states between OP and RP
4. **Periodic Audit Commands** - Maintain synchronization and correct drift

### Compliance Auditing

Compliance scenarios involve OPs auditing RPs to verify proper account management and regulatory compliance without taking over account management responsibilities.

**Audit-Only Sequence**:
1. **Metadata Command** - Establish audit capabilities
2. **Audit Tenant Command** - Review account states and compliance posture
3. **Periodic Audit Commands** - Ongoing compliance monitoring

### Security Incident Response

When security incidents occur, OPs may need to rapidly revoke access across multiple RPs to contain potential breaches.

**Incident Response**:
1. **Invalidate Command** - Immediately invalidate all sessions and tokens for a compromised account
2. **Suspend/Archive Commands** - Temporarily or permanently disable an affected account


# Claims and Properties

This section defines all claims and properties used in this specification. 

## Claims in Command Tokens

- **`aud_sub`**: The RP’s internal identifier for the Account (learned via account resolution). Enables the RP to use its native identifier rather than the (`iss`,`sub`) pair for lookups.
- **`aud`**: Audience for the token; the RP Command Endpoint URL.
- **`authentication_provider`**: A string indicating which party can authenticate the user. Values include:
    - `rp`: Only the RP authenticates
    - `op`: Only the OP authenticates
    - `op_migration`: Both RP and OP can authenticate (temporary migration state)
    - `external`: Authentication is by an external provider
    - `unknown`: Authentication provider is unknown or undisclosed
 
- **`client_id`**: The client identifier for the RP.
- **`command`**: The command name for the RP to execute. Standard values are defined in [Account Commands](#account-commands) and [Tenant Commands](#tenant-commands). Additional values may be defined via [Extensibility](#defining-new-commands).
- **`exp`**: Expiration time (NumericDate, per RFC 7519).
- **`iat`**: Issued at time (NumericDate, per RFC 7519).
- **`iss`**: Issuer Identifier for the OP (see OpenID Connect Core, Section 2).
- **`jti`**: Unique identifier for the token (per OpenID Connect Core Section 9).
- **`metadata`**: A JSON object conveying OP-provided metadata to the RP in the Metadata Command. See [OP Metadata Object](#op-metadata-object) for the normative schema.
- **`sub`**: Subject Identifier for the Account (see OpenID Connect Core, Section 2).
- **`tenant`**: Identifier of the Tenant within the OP. Values include `personal`, `organization`, or an OP-unique stable identifier for multi-tenant OPs. The combination of `iss` and `tenant` identifies a Tenant.



## Properties in Responses

- **`account_state`**: The Account’s state after processing. Supported states include `unknown`, `active`, `suspended`, and `archived`.
- **`aud_sub_required`**: Boolean indicating if the RP requires `aud_sub` in Account Commands.
- **`aud_sub`**: See definition under [Claims in Command Tokens](#claims-in-command-tokens).
- **`authentication_provider`**: See definition and allowed values under [Claims in Command Tokens](#claims-in-command-tokens).
- **`client_id`**: See definition under [Claims in Command Tokens](#claims-in-command-tokens).
- **`command_endpoint`**: The RP’s Command Endpoint URL.
- **`commands_supported`**: Array of command names supported by the RP.
- **`context`**: A JSON object containing contextual values for the response (e.g., `iss`, `tenant`).
- **`error_description`**: Human-readable description of the error.
- **`error`**: Error code indicating the type of error encountered.
- **`last_access`**: NumericDate of last account access (seconds since 1970-01-01T00:00:00Z UTC).
- **`roles`**: Array of role objects with:
    - `id`: RP-unique role identifier
    - `display`: Human-readable role name
    - `description` (optional): Role description
- **`sub`**: See definition under [Claims in Command Tokens](#claims-in-command-tokens).
- **`total_accounts`**: Total number of `account-state` events sent in a streaming response.


For requirement levels for each claim/property in a specific command or response, see the corresponding sections in this specification.


# Command Request

The OP uses an HTTP POST to the registered Command Endpoint
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

OPs issue Commands by sending a JSON Web Token (JWT), the Command Token, to the RP. It is analogous to an OpenID Connect ID Token (see Section 2 of {{OpenID.Core}}) but carries command semantics instead of an authentication assertion.

All claims that can appear in a Command Token are defined once in the [Claims and Properties](#claims-and-properties) section; that section is normative for their definitions. Each command section in this specification normatively states which of those claims are REQUIRED, OPTIONAL, or PROHIBITED for that command. This section therefore only defines the baseline claim groupings used as shorthand elsewhere.

**Prohibition of nonce:** A `nonce` Claim MUST NOT be present in a Command Token to prevent cross-JWT confusion. See [Cross-JWT Confusion](#cross-jwt-confusion).

Baseline claim sets (only the listed claims may appear unless a command explicitly allows additional ones):

- Account Command baseline claims:
  - REQUIRED: `iss`, `aud` (Command Endpoint), `client_id`, `iat`, `exp`, `jti`, `command`, `tenant`, `sub`
  - OPTIONAL: `aud_sub`
 

- Tenant Command baseline claims:
  - REQUIRED: `iss`, `aud` (Command Endpoint), `client_id`, `iat`, `exp`, `jti`, `command`, `tenant`
  - PROHIBITED: `sub`, `aud_sub`
 

Command-specific additions referenced later in this document include (non-exhaustive examples):
- `metadata` — REQUIRED in the Metadata Command; PROHIBITED otherwise
- `authentication_provider` — REQUIRED in the Migrate Command; PROHIBITED otherwise

A Command Token MUST be signed. The same signing keys the OP uses for ID Tokens are used, enabling key discovery via the normal OpenID Connect mechanisms.

Command Tokens MUST be explicitly typed by including a `typ` (type) JWS Header Parameter with the exact value `command+jwt`. See [Security Considerations](#security-considerations) for rationale.

Example JWS header (non-normative):

```json
{
  "alg": "RS256",
  "kid": "2019-07-01-key",
  "typ": "command+jwt"
}
```

A non-normative example JWT Claims Set for the Command Token for an Activate Command follows:

```json
{
  "iss": "https://op.example.org",
  "aud": "https://rp.example.net/command",
  "client_id": "s6BhdRkqt3",
  "iat": 1734003000,
  "exp": 1734003060,
  "jti": "bWJq",
  "command": "activate",
  "sub": "248289761001",
  "given_name": "Jane",
  "family_name": "Smith",
  "email": "jane.smith@example.org",
  "email_verified": true,
  "groups": [
    "b0f4861d",
    "88799417"
  ]
}
```

A non-normative example JWT Claims Set for the Command Token for an Invalidate Command follows:

```json
{
  "iss": "https://op.example.org",
  "aud": "https://rp.example.net/command",
  "client_id": "s6BhdRkqt3",
  "iat": 1734004000,
  "exp": 1734004060,
  "jti": "bWJr",
  "command": "invalidate",
  "sub": "248289761001"
}
```




# Account Commands

Account Commands operate on an Account. Support for any Account Command is OPTIONAL. Account Commands are executed on an RP Account identified in a Command Token by the `aud_sub` claim if provided by the RP during account resolution, or the `iss` and `sub` claims. Account Commands include Lifecycle Commands, the Invalidate Command, and the Migrate Command.

For each Account Command, the Command Token MUST include the Account Command baseline claims defined in [Command Token](#command-token). Each command section below lists only command-specific additions or exceptions. Only the claims listed as REQUIRED or OPTIONAL for a command may be present; all others are PROHIBITED unless otherwise specified.


## Account Lifecycle States

Lifecycle Commands are defined for each transition defined in ISO 24760-1 (2019-05 edition), Section 7.2 for Accounts in an RP's identity register as defined in Section 3.4.5.

Lifecycle Commands transition the Account between the following states:

- **unknown**

- **active**

- **suspended**

- **archived**

Following are the potential state transitions:

```text
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

### Temporary States for Asynchronous Processing

When the RP accepts an Asynchronous Command (responding with HTTP 202 Accepted), the Account MAY enter a temporary “pending” state to indicate that processing is in progress. These temporary states are not steady states in the identity register; they exist only while work is ongoing and are cleared upon completion. The following temporary states are defined:

- `activate_pending`
- `maintain_pending`
- `suspend_pending`
- `reactivate_pending`
- `archive_pending`
- `restore_pending`
- `delete_pending`

Semantics:
- Entry: Upon accepting the corresponding `*_async` command, the RP SHOULD reflect the Account in the matching `*_pending` state.
- Visibility: Pending states MAY be returned wherever `account_state` is surfaced (for example, in Audit responses) to communicate that an operation is in flight.
- Exit (success): When the RP completes processing successfully and sends the asynchronous result to the OP, the Account transitions from `*_pending` to the steady-state outcome of that command (respectively: `active`, `active`, `suspended`, `active`, `archived`, `active`, `unknown`).
- Exit (failure): If the RP cannot complete the operation, the RP MUST clear the pending state and return the Account to its last steady state prior to entering `*_pending`.

## Success Response

When an RP successfully processes an Account Command, the RP returns an `HTTP 200 OK` response and a JSON object containing the following properties:

- `sub` — REQUIRED — the `sub` provided in the request
- `account_state` — REQUIRED — the state of the Account after processing
- `aud_sub` — OPTIONAL — included if the RP has set `aud_sub_required` in its metadata

Following is a non-normative response body to a successful Activate Command:

```json
{
  "account_state": "active",
  "sub": "248289761001"
}
```


## Incompatible State Response

If the Account is in an incompatible state in the identity register for the Account Command, the RP returns an HTTP 409 Conflict and a JSON object containing:

- `account_state` — REQUIRED — the current state of the Account in the identity register
- `error` — REQUIRED — set to the value `incompatible_state`
- `sub` — REQUIRED — the `sub` provided in the request

Following is a non-normative response to an unsuccessful Restore Command where the Account was in the **suspended** state:

```json
{
  "account_state": "suspended",
  "error": "incompatible_state",
  "sub": "248289761001"
}
```

Note that if an Activate Command is sent for an Account that exists, or one of the other Commands are sent for an Account that does not exist, 
the Account is incompatible state. 

Following is a non-normative response to an unsuccessful Activate Command for an existing Account in the **active** state:

```json
{
  "account_state": "active",
  "error": "incompatible_state",
  "sub": "248289761001"
}
```

Following is a non-normative response to an unsuccessful Maintain Command for a non-existing Account:

```json
{
  "account_state": "unknown",
  "error": "incompatible_state",
  "sub": "248289761001"
}
```



## Activate Command
Identified by the `activate` or `activate_async` value in the `command` Claim in a Command Token.

The RP MUST create an Account with the included Claims in the identity register. The Account MUST be in the **unknown** state. The Account is in the **active** state after successful processing.


## Maintain Command
Identified by the `maintain` or `maintain_async` value in the `command` Claim in a Command Token.

The RP MUST update an existing Account in the identity register with the included Claims. The Account MUST be in the **active** state. The Account remains in the **active** state after successful processing.


## Suspend Command
Identified by the `suspend` or `suspend_async` value in the `command` Claim in a Command Token.

The RP MUST perform the [Invalidate Functionality](#invalidate-functionality) on the Account and mark the Account as being temporarily unavailable in the identity register. The Account MUST be in the **active** state. The Account is in the **suspended** state after successful processing.



## Reactivate Command
Identified by the `reactivate` or `reactivate_async` value in the `command` Claim in a Command Token.

The RP MUST mark a suspended Account as being active in the identity register. The Account MUST be in the **suspended** state. The Account is in the **active** state after successful processing. The RP SHOULD support the Reactivate Command if it supports the Suspend Command.


## Archive Command
Identified by the `archive` or `archive_async` value in the `command` Claim in a Command Token.

The RP MUST perform the [Invalidate Functionality](#invalidate-functionality) on the Account and remove the Account from the identity register. The Account MUST be in either the **active** or **suspended** state. The Account is in the **archived** state after successful processing.




## Restore Command
Identified by the `restore` or `restore_async` value in the `command` Claim in a Command Token.

The RP MUST restore an archived Account to the identity register and mark it as being active. The Account MUST be in the **archived** state. The Account is in the **active** state after successful processing. The RP SHOULD support the Restore Command if it supports the Archive Command.


## Delete Command
Identified by the `delete` or `delete_async` value in the `command` Claim in a Command Token.

The RP MUST perform the [Invalidate Functionality](#invalidate-functionality) on the Account, and delete all data associated with an Account. The Account can be in any state except **unknown**. The Account is in the **unknown** state after successful processing.


## Audit Command
Identified by the `audit` or `audit_async` value in the `command` Claim of a Command Token.

The RP MUST include the state of the Account and any Claims for an Account that the RP has retained that were provided by the OP. If the Account is not found, the RP returns `unknown` state.

The response MAY also include:
- `last_access` — OPTIONAL
- `authentication_provider` — OPTIONAL


## Invalidate Command

Identified by the `invalidate` or `invalidate_async` value in the `command` Claim in a Command Token.

The RP MUST perform the [Invalidate Functionality](#invalidate-functionality) on the Account.
The OP MAY send this Command when it suspects a previous OpenID Connect ID Token issued by the OP was granted to a malicious actor, if the user's device was compromised, or any other security related concern about the account. 


The Account MUST be in the **active** state, and remains in the **active** state after executing the Command. If the Account is in any other state, the RP MUST return an [Incompatible State Response](#incompatible-state-response).

The functionality of the Invalidate Command is also performed by Suspend, Archive, and Delete Commands to ensure an Account in the **suspended**, **archived**, and **unknown** state no longer has access to resources.


## Invalidate Functionality

The RP MUST revoke all sessions and tokens including authorization that may have been granted to applications, including `offline_access`, for Account resources identified by the `sub`.


## Migrate Command

Identified by the `migrate` or `migrate_async` value in the `command` claim in a Command Token.

The OP sends this Command to request to become the authentication provider for an existing Account. The Command Token MUST include the following additional claim:

- **authentication_provider**  
  REQUIRED.  
  A JSON string indicating the requested authentication provider status. MUST be set to `op` if the OP wants to become the sole authentication provider for the Account, or `op_migration` if the OP wants to enable dual authentication where both the OP and existing authentication methods remain valid.

The RP processes the Migrate Command based on the current authentication provider status of the Account and organizational policies:

- If `authentication_provider` is `op` and the RP allows the migration, the RP updates the Account's authentication provider status to `op`, disabling other authentication methods.
- If `authentication_provider` is `op_migration` and the RP supports dual authentication, the RP adds the OP as an additional authentication provider while maintaining existing authentication methods. Dual authentication with `op_migration` SHOULD be a temporary state used to provide a seamless migration from RP to OP as the authentication provider.

### Access Denied Error

If the RP does not allow the OP to migrate authentication for the Account, the RP MUST return an HTTP 403 Forbidden response and include the `error` parameter with the value of `access_denied`.

### Authentication Not Transferable Error

If the Account's current authentication provider does not allow changes in authentication responsibility (e.g., managed by a different OP or external system with non-transferable policies), the RP MUST return an HTTP 409 Conflict response and include the `error` parameter with the value of `authentication_not_transferable`.



# Tenant Commands

Tenant Commands are executed in the context of a Tenant. The RP MUST support the Metadata Command. Support for other Tenant Commands is optional. Command Tokens for Tenant Commands MUST contain the `tenant` claim, and are PROHIBITED from containing the `sub` claim. See [Claims and Properties](#claims-and-properties) for definitions.

Note that no Asynchronous Commands are defined to execute in the context of a Tenant.


## Metadata Command

Identified by the `metadata` value in the `command` claim in a Command Token.

Additional Command Token claims:
- `metadata` — REQUIRED — OP-provided metadata object

The OP sends this Command to exchange metadata with the RP. The OP sends its metadata in the Command Request, and the RP sends its metadata in the Command Response. A Metadata Command replaces any previous metadata provided by the OP to the RP in the Command Request, and any metadata provided by the RP to the OP in the Command Response.

### OP Metadata Object

The `metadata` claim in the Metadata Command contains a JSON object with the following fields:

- `notification_endpoint` — OPTIONAL — HTTPS URL for the OP notification endpoint that RPs can use to send notifications.
- `domains` — OPTIONAL — Array of strings. Email or DNS domains associated with the Tenant (for discovery, correlation, or scoping at the RP).
- `claims_supported` — OPTIONAL — Array of strings. Claim names the OP may include in Account Commands or expect back in responses.
- `groups` — OPTIONAL — Array of group objects. Each object has:
  - `id` — REQUIRED — OP-unique group identifier
  - `display` — REQUIRED — Human-readable group name
  - `description` — OPTIONAL — Group description
 
Additional fields MAY be included, including those defined in OAuth 2.0 Authorization Server Metadata [RFC8414](#RFC8414). Unknown fields MUST be ignored by recipients.

Following is a non-normative example of a Claim set in a Command Token for the Metadata Command:

```json
{
  "iss": "https://op.example.org",
  "aud": "https://rp.example.net/command",
  "client_id": "s6BhdRkqt3",
  "iat": 1734006000,
  "exp": 1734006060,
  "jti": "bWJt",
  "command": "metadata",
  "tenant": "ff6e7c96",
  "metadata": {
    "notification_endpoint": "https://op.example.org/callback",
    "groups": [
      {
        "id": "b0f4861d",
        "display": "Administrators",
        "description": "Application administrators"
      },
      {
        "id": "88799417",
        "display": "Finance",
        "description": "Everyone in corporate finance"
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

- `context` (REQUIRED): A JSON object containing `iss` and `tenant` values from the Command Token.
- `commands_supported` (REQUIRED): A JSON array of Commands the RP supports. The `metadata` value MUST be included.
- `command_endpoint` (REQUIRED): The RP's Command Endpoint. This is the URL the Command Token was sent to.
- `client_id` (REQUIRED): The `client_id` for the RP.

The response MAY also include:
- `aud_sub_required` (OPTIONAL): Indicates the RP requires its `aud_sub` value be provided in account commands.
- `roles` (OPTIONAL): A JSON array of objects describing roles supported by the RP.
- `jwks_uri` (OPTIONAL): A URL pointing to a set of JSON-encoded public keys, represented as a JWK Set [RFC7517], which the RP may use for signing notifications sent to the OP. 
 

The response MAY also include any OAuth Dynamic Client Registration Metadata *TBD [IANA reference](https://www.iana.org/assignments/oauth-parameters/oauth-parameters.xhtml#client-metadata)*

The RP MAY provide a different response for different `iss` or `tenant` values.

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
  "command_endpoint": "https://rp.example.net/command",
  "commands_supported":[
    "audit_tenant",
    "unauthorize",
    "suspend",
    "reactivate",
    "delete",
    "audit"
  ],
  "claims_supported": [
    "sub",
    "email",
    "email_verified",
    "name",
    "roles"
  ],
  "roles": [
          {
        "id": "00001",
        "display": "Admins",
        "description": "All administrative"
      },
      {
        "id": "00002",
        "display": "Editors",
        "description": "Create, read, update, delete posts"
      },
      {
        "id": "00003",
        "display": "Reader",
        "description": "Read posts"
      }
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

All Tenant Commands besides the Metadata Command use [Server-Sent Events](https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events) (SSE) for the RP to transfer the Command Response. When performing a Streaming Request, the OP MUST include the following HTTP headers when sending the Command Request:

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

The RP sends a Streaming Response to a Streaming Request. In a Streaming Response, the RP uses SSE to stream the Command Response as a sequence of events. If the RP receives a valid Command, it MUST sent the `HTTP 200 OK` response, followed by the following headers:

- `Content-Type` with the `text/event-stream` value
- `Cache-Control` with the `no-cache` value
- `Connection` with the `keep-alive` value

If the OP sent a `Content-Encoding` header in the request with a compression the RP understands, the RP MAY include a `Content-Encoding` header with one of the OP provided values.

Per SSE, the body of the response is a series of events. In addition to the required field name `data`, each event MUST include the `id` field with a unique value for each event, and the `event` field with a value of either `account-state`, `command-complete`, or `error`. The RP sends an `account-state` event for each Account at the RP for the `iss`, and `org` if sent, in the Audit Tenant Command. When all `account-state` events have been sent, the RP sends a `command-complete` event. If an unrecoverable error occurs during processing, the RP SHOULD send an `error` event where the `data` field is JSON with the JSON string property `error_description` set to a value determined by the RP.

The `data` parameter of the `account-state` event MUST contain: `sub`, `account_state`.

The `data` parameter MAY include other Claims as defined by the Tenant Command.

The `data` parameter of the `command-complete` event MUST include `total_accounts` with the total number of `account-state` events the RP has sent.

If there are no Accounts for the Tenant at the RP, the RP responds with only the `command-complete` event with `total-accounts` having a value of `0`.


The following is a non-normative example of a Streaming Response for an Audit Tenant Command:

```bash
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
    "account_state": "active",
    "last_access": 1746792000,
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


If the connection is lost during a Streaming Response, The OP SHOULD generate a new Command Token and send a Streaming Request including the HTTP header `Last-Event-Id` with the last event `id` property received per SSE. 

Following is a non-normative example of a Streaming Request sent after a connection was lost:

```bash
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

If the RP is unable to resume a Streaming Response when provided a `Last-Event-Id` HTTP header, it MUST respond with an HTTP 404 Not Found and include the `error` parameter of `last-event-id-unavailable`.

## Audit Tenant Command

Sent in a Streaming Request and identified by the `audit_tenant` value in the `command` Claim in a Command Token.

The OP sends the Audit Tenant Command to learn the state of Accounts for a Tenant at an RP. 

The following is a non-normative example of the Claims Set in the Command Token of an Audit Tenant Command:

```json
{
  "iss": "https://op.example.org",
  "aud": "https://rp.example.net/command",
  "client_id": "s6BhdRkqt3",
  "iat": 1734003000,
  "exp": 1734003060,
  "jti": "bWJz",
  "command": "audit_tenant"
}
```


## Audit Tenant Response

The RP sends a Streaming Response if it received a valid Audit Tenant Command.

The RP MUST include any Claims for an Account that the RP has retained that were provided by the OP in the event `data` parameter JSON string. If the Claim values have been modified at the RP, the modified values should be returned. 

The RP MAY include a `last_access` claim, a NumericDate, representing the number of seconds from 1970-01-01T00:00:00Z UTC. The value MUST be an integer and is equivalent to the iat and exp claims as defined in [RFC7519](#RFC7519).

Streaming Response Events:
- `account-state` events MUST include: `sub` (REQUIRED), `account_state` (REQUIRED)
- `account-state` events MAY include: `last_access`, `authentication_provider`, other retained Claims
- A final `command-complete` event MUST include: `total_accounts` (REQUIRED)


## Suspend Tenant Command

Sent in a Streaming Request and identified by the `suspend_tenant` value in the `command` Claim in a Command Token.
Upon receiving this Command, the RP MUST suspend all Accounts in the active state for the Tenant identified by the `tenant` Claim in the Command Token. 

The RP sends a Streaming Response if it received a valid Suspend Tenant Command.


Streaming Response Events:
- `account-state` events MUST include: `sub` (REQUIRED), `account_state` (value `suspended`)
- A final `command-complete` event MUST include: `total_accounts` (REQUIRED)

## Archive Tenant Command

Sent in a Streaming Request and identified by the `archive_tenant` value in the `command` Claim in a Command Token.
Upon receiving this Command, the RP MUST suspend all Accounts in the active and suspended state for the Tenant identified by the `tenant` Claim in the Command Token.

The RP sends a Streaming Response if it received a valid Archive Tenant Command.


Streaming Response Events:
- `account-state` events MUST include: `sub` (REQUIRED), `account_state` (value `archived`)
- A final `command-complete` event MUST include: `total_accounts` (REQUIRED)


## Delete Tenant Command

Sent in a Streaming Request and identified by the `delete_tenant` value in the `command` Claim in a Command Token.
Upon receiving this Command, the RP MUST delete all Accounts for the Tenant identified by the `tenant` Claim in the Command Token.

The RP sends a Streaming Response if it received a valid Delete Tenant Command. If the Delete Tenant Command was successful, the RP will send only a `command-complete` event with a `data` parameter containing the JSON string `{"total_accounts":0}`.


Streaming Response Events:
- A final `command-complete` event MUST include: `total_accounts` (REQUIRED). No `account-state` events are sent upon successful deletion of all Accounts.

## Invalidate Tenant Command

Sent in a Streaming Request and identified by the `invalidate_tenant` value in the `command` Claim in a Command Token.
Upon receiving this Command, the RP MUST perform the [Invalidate Functionality](#invalidate-functionality) on all Accounts in the **active** state for the Tenant identified by the `tenant` Claim in the Command Token.

The RP sends a Streaming Response if it received a valid Invalidate Tenant Command.


Streaming Response Events:
- `account-state` events MUST include: `sub` (REQUIRED), `account_state` (value `active`)
- A final `command-complete` event MUST include: `total_accounts` (REQUIRED)


# Notifications


Notifications enable an RP to notify an OP of a change in the RP state or the completion of an asynchronous command. Notifications are authenticated using a Notification Token issued and signed by the RP.

## Notification Token

A Notification Token is a JSON Web Token (JWT) signed by the RP. The OP validates the signature using the RP's published keys (e.g., via the `jwks_uri` value that the RP provided in the Metadata Response). To avoid cross-JWT confusion, Notification Tokens MUST include a `typ` JWS header of `notification+jwt`.

Non-normative JWS header example:

```json
{
  "alg": "RS256",
  "kid": "2025-09-01",
  "typ": "notification+jwt"
}
```

Notification Tokens are posted to the OP's `notification_endpoint` using `application/x-www-form-urlencoded` as the HTTP request body and MUST include a parameter named `notification_token` whose value is the JWS Compact Serialization of the token.
The OP MUST respond with HTTP 204 No Content on success. Error responses use standard error status codes and MAY include a JSON body with `error` and `error_description`.

```
POST /notifications HTTP/1.1
Host: op.example.org
Content-Type: application/x-www-form-urlencoded
Accept: application/json
Cache-Control: no-cache

notification_token=eyJhbGciOiJS...
```


## Notification Claims


All Notification Tokens MUST include the following claims in the payload:

- `iss` - the RP 
- `aud` - the OP issuer value
- `client_id` - the RP's client_id
- `iat` - 
- `jti` -
- `tenant` - the OP tenant
- `notification` - the type of notification (see Notification Types)


If the notification is for the result of an async command, the properties of the async command will also be included as claims.

### Notification Types

The `notification` claim in the Notification Token payload indicates the type of notification. This specification defines the following values:

- `metadata_change` — RP indicates its metadata has changed. The OP SHOULD initiate a new Metadata Command.
- `audit_tenant_change` — RP indicates there have been changes to the tenant’s accounts. The OP SHOULD initiate a new Audit Tenant Command.
- `*_async_response` — RP is sending the final result for an asynchronous command. The asterisk form is a placeholder for the command name (for example: `activate_async_response`, `suspend_async_response`). 

Additional values MAY be defined by future extensions. The OP MUST ignore unknown notification values.

### Examples

Non-normative examples of Notification Token payloads (JWT Claims Sets):

- metadata_change

```json
{
  "iss": "https://rp.example.net",
  "aud": "https://op.example.org",
  "client_id": "s6BhdRkqt3",
  "iat": 1734007000,
  "jti": "nt-001",
  "tenant": "ff6e7c96",
  "notification": "metadata_change"
}
```

- activate_async_response (successful)

```json
{
  "iss": "https://rp.example.net",
  "aud": "https://op.example.org",
  "client_id": "s6BhdRkqt3",
  "iat": 1734007100,
  "jti": "nt-002",
  "tenant": "ff6e7c96",
  "notification": "activate_async_response",
  "sub": "248289761001",
  "account_state": "active"
}
```

- suspend_async_response (unsuccessful)

```json
{
  "iss": "https://rp.example.net",
  "aud": "https://op.example.org",
  "client_id": "s6BhdRkqt3",
  "iat": 1734007200,
  "jti": "nt-003",
  "tenant": "ff6e7c96",
  "notification": "suspend_async_response",
  "sub": "248289761001",
  "account_state": "archived",
  "error": "incompatible_state",
  "error_description": "Account is archived and cannot be suspended"
}
```


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

## Defining New Notifications

*To be completed.*


# OpenID Provider Command Support

Relying Parties supporting OpenID Provider Commands register a Command Endpoint with the OP as part of their client registration.

The Command Endpoint MUST be an absolute URI as defined by
Section 4.3 of {{RFC3986}}.
The Command Endpoint MAY include an
`application/x-www-form-urlencoded` formatted
query component, per Section 3.4 of {{RFC3986}}.
The Command Endpoint MUST NOT include a fragment component.

If the RP supports
[OpenID Connect Dynamic Client Registration 1.0](#OpenID.Registration),
it uses this metadata value to register the OpenID Provider Command Endpoint:

- **command_endpoint**  
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

- **Client Metadata Name:** `command_endpoint`
  
**Client Metadata Description:**
  RP URL that will receive OpenID Provider Commands from the OP

**Change Controller:** OpenID Foundation

**Specification Document(s):** This document


## Media Type Registration

This specification registers the `application/command+jwt` media type as per {{!RFC6838}}.

*To be completed.*


# References

## Normative References

- **[RFC2119]** Bradner, S. “Key words for use in RFCs to Indicate Requirement Levels,” *RFC 2119*, March 1997.
- **[RFC3986]** Berners-Lee, T., Fielding, R., and Masinter, L. “Uniform Resource Identifier (URI): Generic Syntax,” *RFC 3986*, January 2005.
- **[RFC6750]** Jones, M., and Hardt, D., “The OAuth 2.0 Authorization Framework: Bearer Token Usage,” *RFC 6750*, October 2012
- **[RFC7519]** Jones, M., Bradley, J., and Sakimura, N. “JSON Web Token (JWT),” *RFC 7519*, May 2015.
- **[RFC7591]** Jones, M., Bradley, J., and Sakimura, N. “OAuth 2.0 Dynamic Client Registration Protocol,” *RFC 7591*, March 2015.
- **[RFC8414]** Jones, M., and Sakimura, N., “OAuth 2.0 Authorization Server Metadata,” *RFC 8414*, June 2018.
- **[RFC9110]** Fielding, R. “HTTP Semantics,” *RFC 9110*, June 2022.
- **[RFC8725]** Bromberg, L. “Security Considerations for JSON Web Tokens,” *RFC 8725*, June 2020.
- **[RFC6838]** IANA. “Media Types,” *RFC 6838*, June 2013.
- **ISO/IEC 24760-1:2019**, “IT Security – A framework for identity management – Part 1: Terminology and concepts.”
- **OpenID.Core** – “OpenID Connect Core 1.0 incorporating errata set 2,” available at <https://openid.net/specs/openid-connect-core-1_0.html>.
- **OpenID.Enterprise** – "OpenID Connect Enterprise Extensions 1.0," available at <https://openid.net/specs/openid-connect-enterprise-extensions-1_0.html>.

## Informative References

- **IANA JSON Web Token Claims Registry**, available at <https://www.iana.org/assignments/jwt/jwt.xhtml>.
- **IANA OAuth Parameters**, available at <https://www.iana.org/assignments/oauth-parameters/oauth-parameters.xhtml#client-metadata>.

{backmatter}

# Acknowledgements

The authors would like to thank early feedback provided by Tim Cappalli, Andrii Deinega, Pam Dingle, George Fletcher, Michael Jones, Jeff Lombardo, Aaron Parecki, Nat Sakimura, Dean Saxe, and Rifaat Shekh-Yusef.

*To be updated.*

# Notices

Copyright (c) 2025 The OpenID Foundation.

The OpenID Foundation (OIDF) grants to any Contributor, developer,
implementer, or other interested party a non-exclusive, royalty free,
worldwide copyright license to reproduce, prepare derivative works from,
distribute, perform and display, this Implementers Draft, Final
Specification, or Final Specification Incorporating Errata Corrections
solely for the purposes of (i) developing specifications,
and (ii) implementing Implementers Drafts, Final Specifications,
and Final Specification Incorporating Errata Corrections based
on such documents, provided that attribution be made to the OIDF as the
source of the material, but that such attribution does not indicate an
endorsement by the OIDF.

The technology described in this specification was made available
from contributions from various sources, including members of the OpenID
Foundation and others. Although the OpenID Foundation has taken steps to
help ensure that the technology is available for distribution, it takes
no position regarding the validity or scope of any intellectual property
or other rights that might be claimed to pertain to the implementation
or use of the technology described in this specification or the extent
to which any license under such rights might or might not be available;
neither does it represent that it has made any independent effort to
identify any such rights. The OpenID Foundation and the contributors to
this specification make no (and hereby expressly disclaim any)
warranties (express, implied, or otherwise), including implied
warranties of merchantability, non-infringement, fitness for a
particular purpose, or title, related to this specification, and the
entire risk as to implementing this specification is assumed by the
implementer. The OpenID Intellectual Property Rights policy
(found at openid.net) requires
contributors to offer a patent promise not to assert certain patent
claims against other contributors and against implementers.
OpenID invites any interested party to bring to its attention any
copyrights, patents, patent applications, or other proprietary rights
that may cover technology that may be required to practice this
specification.

# Document History

   [[ To be removed from the final specification ]]

   -00

   initial draft

   -01

   * rename `commands_uri` to `commands_endpoint`
   * added `audit` account command
   * change `aud` to be the `commands_endpoint` and add `client_id` as a separate claim
   * add `error` as another event type in tenant SSE response
   * add async commands, `notification_endpoint` to OP metadata, and `callback_token` for async response and `metadata` request 
   * add error messages for restarting a stream that cannot be restarted
   * add `roles` claims and `roles` metadata
   * add `last_access` claim for RP to communicate last time user accessed the resource off_line 
   * minor edits and structural cleanup

  -02

  * added `aud_sub` as claim in commands and property from RP in audit reponses
  * Metadata Response: Added `aud_sub_required` response property (OPTIONAL) normatively indicating RP requirement to receive `aud_sub` in subsequent Account Commands.
  * collected all normative claims and properties into new "Claims and Properties" section centralizing definitions of all Command Token claims and response properties
  * Command Token: Introduced normative baseline claim sets for Account vs Tenant Commands; clarified that only listed claims plus command-specific additions may appear (tightening allowed claims surface).
