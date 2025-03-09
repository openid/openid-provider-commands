# FAQ 

1. How does SCIM compare to OpenID Provider (OP) Commands?

The SCIM protocol is a general purpose protocol for a client to manage resources at a server. When the SCIM protocol is used between an IdP and an RP, the schema is defined by the RP. The resources managed are in the context of the RP Tenant in a multi-tenant RP. Any extensions to the schema are defined by the RP. This provided an interoperable protocol to manage RP resources. OpenID Provider Commands are an extension of a user Account created by OpenID Connect. It uses the same identity Claims that the OP issues for the user. It uses the same token Claims, and is verified the same way. OpenID Provider Commands are issued in the context of the OP Tenant in a multi-tenant OP.

2. How do Shared Signals / RISC compare to OpenID Provider Commands?

Shared Signals and RISC are events that one party is sharing with another party. The actions a receiving party takes upon receiving a signal are intentionally not defined. The actions taken by the RP when receiving an OpenID Provider Command is specified. This gives an OP control over the Account at the RP.

3. Are OpenID Provider Commands a replacement for SCIM, Shared Signals, or RISC?

No. These standards are deployed by organizations that have complex requirements, and these standards meet there needs. Most OP / RPs do not deploy any of these standards, as the implementation complexity is not warranted. OpenID Provider Commands are designed to build on OpenID Connect, allowing RPs using OpenID Connect an easy path to offer OPs a standard API for security and lifecycle operations.

4. Why are there only groups? Why not roles and entitlements?

OpenID Provider Commands are used to project the Tenant data managed centrally by the OP. Groups are a common term used by OPs to manage a collection of Accounts. The terms roles and entitlements tend to be RP specific. Generally, groups from the OP will be mapped to roles and/or entitlements that are RP specific, at the RP.