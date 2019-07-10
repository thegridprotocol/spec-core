# The Grid Protocol - Fundamentals

The following RFCs are prerequisites to this document:

- [2119](https://tools.ietf.org/html/rfc2119)  - *Key words for use in RFCs to Indicate Requirement Levels*
- [2606](https://tools.ietf.org/html/rfc2606) - *Reserved Top Level DNS Names*
- [8174](https://tools.ietf.org/html/rfc8174) - *Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words*

This document is to be considered a diff of the Matrix 1.0 spec which this project forked. Anything not documented is to be taken from the Matrix specs with naming adaptation to IDs, namespaces and types.

The following versions of the APIs are used:

| **API**             | **Version**                                                  |
| ------------------- | ------------------------------------------------------------ |
| Client-Server       | [r0.5.0](https://matrix.org/docs/spec/client_server/r0.5.0)  |
| Server-Server       | [r0.1.2](https://matrix.org/docs/spec/server_server/r0.1.2)  |
| Application Service | [r0.1.1](https://matrix.org/docs/spec/application_service/r0.1.1) |
| Identity Service    | [r0.2.1](https://matrix.org/docs/spec/identity_service/r0.2.1) |
| Push Gateway        | [r0.1.0](https://matrix.org/docs/spec/push_gateway/r0.1.0)   |

## Concepts

### Roles

The Grid protocol specifies *roles* that implementations can take and APIs linked to those roles. The current defined roles are:

- Client: Interacting with users (human or not) and acting as a gateway to the server functions.
- Server: Interacting with other servers, acting as a network node to achieve the decentralised approach of The Grid.

An implementation can be either or both. This specification makes no assumption on implementations implementing the role(s) in a particular way.

The Grid protocol takes no position on how decentralisation is achieved. [Peer-to-Peer](https://en.wikipedia.org/wiki/Peer-to-peer) and [Federation](https://en.wikipedia.org/wiki/Federation_(information_technology)) are typical ways, the first having one implementation being both client and server while the second separates the two roles into their dedicated implementations.

Implementations are free to take on whichever approach they feel is appropriate for their needs. This specification will attempt to not artificially restrict possible use cases and remain as open as possible. Developers are encouraged to report any artificial restrictions they encounter so the protocol technical board can discuss if such restriction should be taken care of or not.

### Realm

A `realm` is the value under which [Identifiers](#identifiers) are namespaced. A `realm` is considered an opaque string unless specified otherwise.

### Events

Events are fundamental unit of data exchanged between servers over [Channels](#channels).

#### Types

There are four fundamental types of events:

| Type                              | Has an ID? | Has Parent Events? |
| --------------------------------- | ---------- | ------------------ |
| Persistent Linked Event (`PLE`)   | Yes        | Yes                |
| Persistent Unlinked Event (`PUE`) | Yes        | No                 |
| Ephemeral Linked Event (`ELE`)    | No         | Yes                |
| Ephemeral Unlinked Event (`EUE`)  | No         | No                 |

> **TODO**: Link to specific usages of those types in the spec, once the various sections are complete enough.
>
> **NOTE:** Format follows Matrix for now, except for `auth_events` not being kept and `state_key` being renamed to `scope`

#### Ordering

In the context of a channel, two types of ordering are defined:

- `DAG order`
- `Timeline order`

In both case, ordering is always based on the relationships between events which give a *happen-before* guarantee. Any time-based key like `timestamp` **MUST NOT** be trusted for any kind of ordering used in state computation/resolution or performing any kind of authoritative action/computation.

Time-based keys are only informational, and **MAY** be used for user experience/presentation purposes.

##### DAG order

DAG order is always a relative ordering of a subset of a channel. The subset is between an event and its recursive parents, optionally scoped to depth `N`. This ordering doesn't include any side branch of the DAG.

It is formally defined as such for an event `O` located anywhere in the DAG, to collect `O` parents recursively, called individually `P` , then stop processing across all branches if `P` depth is equal or lower than `N`, or `P` has no parents. Once all events are collected, order using the following:

- By descending `depth` value
- If the same, by the presence of a non-null `scope` key, with order  `withKey, withoutKey`
- If the same, by ascending lexical order of the `id` value which must be unique per channel

The ordering is naturally in going backward.

> **TODO**: Define
>
> - `parent`
> - `recursive parent`
> - `backward`
> - `forward`

##### Timeline order

Across all branches, Is ascending order. Is the "human" view.

> **TODO**: Document the exact algorithm in details.

##### Examples

Given the following example structure:

```
         A       |  (Depth = 1337)
         |       |
         B       |
      /  |  \    |
     C   D   E   F
       \ |  /    |
         G       H
         |       |
```

Knowing that:

- The `depth` of `A` is `1337`
- `A` ID is `$a` and has no `scope` key
- `B` ID is `$b` and has no `scope` key
- `C` ID is `$c` and has no `scope` key
- `D` ID is `$d` and has a `scope` of value `@b`
- `E` ID is `$e` and has no `scope` key
- `F` ID is `$f` and has a `scope` of value `@a`. `F` has parent(s) with a `depth` lower than `1337`
- `G` ID is `$g` and has no `scope` key
- `H` ID is `$h` and has no `scope` key

The DAG order from `G` with limit of `1337` for `N` would be: `G, D, E, C, B, A`.

The Timeline order from `A` with limit of `1340` for `N` would be: `A, B, F, D, C, E, G, H`.

### Channels

[Events](#events) are exchanged over a fundamental structure called `channel`. Channels are uniquely identified with a unique ID. Channels **MAY** be referenced by one to many aliases. A channel **MAY** not have aliases.

## Exchanges

The protocol aims to promote the most recent technologies that are stable enough or will be stable enough, while not putting an unreasonable burden on developers to build software based on the protocol.

Therefore, we aim that all implementations should support the following combined set of technologies as soon as possible:

- Data representation: `JSON`
- Data encoding: `UTF-8`
- Transport: `HTTP/3`
- Security: `TLS 1.3`

While keeping this in mind, the following sections define the precises rules and fallback mechanisms for each area.

### Data

[JSON](https://en.wikipedia.org/wiki/JSON) ([RFC 8259](https://tools.ietf.org/html/rfc8259)) is the baseline and default data format that **MUST** be supported by all implementations - MIME type is `application/json`. 

> **Rationale**: We follow on the founding documents stating the first pieces of spec work should be based on the Matrix protocol.

### Encoding

[UTF-8](https://en.wikipedia.org/wiki/UTF-8) ([RFC 3629](https://tools.ietf.org/html/rfc3629)) is the baseline and default data encoding that **MUST** be supported by all implementations.

> **Rationale**: We follow on the founding documents stating the first pieces of spec work should be based on the Matrix protocol.

When exchanging data between implementations, if there is a possiblity to specify the MIME Type, Implementations **SHOULD NOT** append a charset like `; charset=utf-8` if the encoding is `UTF-8`.

>  **Rationale**: Keep it straight-forward for implementations.

### Transport

[HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol) is the baseline and default transport to be supported by all implementations. Implementations **SHOULD** use [HTTP/2](https://en.wikipedia.org/wiki/HTTP/2) ([RFC 7540](https://tools.ietf.org/html/rfc7540)) by default and only fallback to HTTP/1.1 ([RFC 7230](https://tools.ietf.org/html/rfc7230)) if HTTP/2 is not available. Any prior version **SHOULD NOT** be used.

>  **Rationale**: Using `HTTP` as we follow on the founding documents stating the first pieces of spec work should be based on the Matrix protocol.
>
>  Dropping support for `HTTP/1.0` and prior to follow on *Exchanges* goals.
>
>  Promote `HTTP/2` now that it is a stable RFC to promote acceptance.
>
>  `HTTP/3` is not a mature specification yet, and might burden implementations too much at this stage of the protocol if used as the default transport. However, once the specification matures to the point of being acceptable for wider use, implementations are encouraged to utilize it as the default transport, as soon as possible, with fallback to `HTTP/2` and `HTTP/1.1` as necessary.
>
>  `HTTP/3`, `HTTP/2` and `HTTP/1.1` - in this order - allow for fallback, making it seamless to users. The fallback is normally largely adopted in libraries and SDKs, also making it seamless for developers.


>  **Open Question**: While the *Exchanges* section talks about `HTTP/3` as a wanted goal, we don't talk about it here specifically - **How can we word this better?**

### Security

[TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security) [1.2](https://en.wikipedia.org/wiki/Transport_Layer_Security#TLS_1.2) ([RFC 5246](https://tools.ietf.org/html/rfc5246)) is the baseline and default security layer for HTTP. Implementations that provide a mean to connect over any kind of public network **MUST** support it and use it by default.

> **Rationale**: Enforce the privacy value of the project and follow on the founding documents stating the first pieces of spec work should be based on the Matrix protocol.

Implementations **SHOULD** support [TLS 1.3](https://en.wikipedia.org/wiki/Transport_Layer_Security#TLS_1.3) ([RFC 8446](https://tools.ietf.org/html/rfc8446)).

> **Rationale**: Follow *Exchanges* section goals.

## Identifiers

Identifiers will be split into two categories:

- `ID`: Opaque low-level Identifier used by implementations to uniquely identify and optionally route data.
- `Alias`: High-level Identifier to be created and used by users.

### IDs

An `ID` is defined as a compound string made of a single character in first position, called `sigil` followed by a globally unique network identifier. The generation algorithms are specific to each `ID` type.

An `ID` **MUST** be treated as an opaque, case-sensitive string and used as-is unless a specific mechanism requires it to be interpreted. An `ID` is considered NOT human/users friendly and not to be used by them. They are to be used directly by software. Implementations of the protocol **SHOULD NOT** promote their usage by users outside of issue reporting, troubleshooting or advanced operations.

> **Open Question**: *Opaque* is not defined in a technical manner, and may be unclear and/or misleading.
> **Is there a better word that states that implementations must not try to make sense of it, unless specifically stated in this doc?**


> **Rationale**: an `ID` is a low-level identifier and is therefore not meant to be used by humans/users directly. Given the privacy goal of the protocol, it is also important to split the concepts of `ID` not inherently tied to an identifiable information and `alias` which is specifically created by a user/system so an entity can specifically be identified.
>
> Making an `ID` opaque and case-sensitive allows for them to be treated as bytes without putting any kind of restriction for the future, while their companion Identifier, `aliases` deal with user-specific needs.

Except for the `sigil` defining the ID type, characters **MUST** be from the [Base64](https://tools.ietf.org/html/rfc4648) encoding without padding and use the [URL-safe variant](https://tools.ietf.org/html/rfc4648#section-5).

> **Rationale**:
>
> *Without padding*: keep IDs as small as possible, as they will repeat in each event.
>
> *Base64 encoding*:
>
> - Given the opaque and "bytes-like" nature of an `ID`, Base64 makes it safe across virtually all possible transports.
> - Gives the ability to "hide" regular text from users. This particular issue was experienced when working on Matrix where users regularly asks why Identifiers like Room IDs cannot be used to join them since they contain the server hostname that originally created the room.
>
> *URL-safe variant*: avoid any mess in URLs with things like `/` and avoid implementations bugs that still treat `/` and other similar characters as special characters even after decoding.

Example:

```
@fdsD-5_43
```

- `@` is the `sigil`.
- `fdsD-5_43` is the globally unique network identifier.

#### Ecosystem bootstrap

To bootstrap the ecosystem, IDs **MUST** be built like this:

1. Pick an arbitrary unique identifier for the object, unique within the configured realm.
2. Append `@`
3. Append the Grid realm
4. Encode as Base64 URL-safe no-padding
5. Add ID `sigil`

Example:

For the user `john` on the domain `example.org`:

1. `john`
2. `john@`
3. `john@example.org`
4. `am9obkBleGFtcGxlLm9yZwo`
5. `@am9obkBleGFtcGxlLm9yZwo`

For this version of the spec, Implementations **MUST** decode IDs in reverse to be able to route messages and initiate connections.

> **Rationale**: While the protocol will tackle decentralised identifiers, identity and overall privacy-protecting measures, it is important to have a migration path from Matrix and current DNS systems to a new system.
>
> This bootstrap measure allows to:
>
> - Use currently existing environments/setups to bootstrap Grid
> - Trivially hide things like DNS hostnames/domains in a way which is fully compatible with any future choice
> - Pave the way to Identity mappings and features like account migration at virtually no extra cost, making all these fundamentally "built-in" as soon as the Identity part of the specification is produced.

#### Servers

**Sigil:** `:`

#### Users

**Sigil:** `@`

#### Channels

**Sigil:** `#`

#### Events

**Sigil:** `$`

### Aliases

An `alias` is defined as a compound string made of four elements, in order:

- A `sigil` to define the type of `alias`
- A human-friendly identifier/name, unique within the [realm](#realm) it was created
- The character `@` 
- A Grid [realm](#realm)

#### Users

**Sigil:** `@`

Example:

```
@john@example.org
```

#### Channels

**Sigil:** `#`

Example:

```
#grid@example.org
```

## APIs

### Security

Access to endpoints **MAY** be restricted by the use of credentials from the client to the server.

#### Authentication

Authentication in Client APIs is User-Interactive based. This model follows the [Matrix Client API r0.5.0 specification](https://matrix.org/docs/spec/client_server/r0.5.0#user-interactive-authentication-api) §5.3 except for the following sections which are redefined in this specification:

- The authentication types: §5.3.4, §§5.3.4.1-7
- The fallback mechanism: §5.3.5
- The identifier types: §5.3.6

> **NOTE:** The mechanism will be documented in full in this specification by v0.1.0, while following our funding documents to base this protocol on Matrix.

All endpoints allowing credentials are eligible for further, isolated User-Interactive authentication.

##### Types

Authentication in The Grid is fundamentally open and allows the use of custom types for specific uses with a fallback mechanism allowing their use regardless of client support.

The protocol will define a set of most commonly used authentication methods and in which API they can be used when requiring specific flows.

Servers **MUST** provide a fallback of type `g.fallback.web` for all and any Authentication types that can be requested.

###### Password

**Type:** `g.auth.password`
**API:** Identity

Matches Matrix § 5.3.4.1

> **NOTE:** Will be fully documented in v0.1.0

###### Token

**Type:** `g.auth.token`
**API:** *Any*

Tokens are one-off opaque strings given to a user to allow a single use of the endpoint. Tokens **MUST** be invalidated after being used once.

Typical uses include:

- Invitation code allowing users to register on a server which does not allow public registrations.
- Validate ownership of an Email or a Phone number.
- Registration of an external service.

The following response keys are available:

| **Key** | **Mandatory** | **Purpose**                  |
| ------- | ------------- | ---------------------------- |
| `token` | **Yes**       | To be set to the token value |

Example:

```json
{
    "session": "RandomSessionID",
    "type": "g.auth.token",
    "token": "<Token to give to the server>"
}
```

###### Signature

**Type:** `g.auth.sign.ed25519`
**API:** *Any*

The following request parameters are available:

| **Key** | **Type** | **Required** | Purpose                                                |
| ------- | -------- | ------------ | ------------------------------------------------------ |
| `key`   | String   | **Yes**      | The public key identifier that needs authentication.   |
| `data`  | String   | **Yes**      | The data to sign to proof ownership of the public key. |

The following response keys are available:

| Key    | **Type** | **Required** | Purpose                                             |
| ------ | -------- | ------------ | --------------------------------------------------- |
| `sign` | String   | **Yes**      | The signature proving ownership of the private key. |



##### Identifier Types

The following official types are available:

- `g.id.username`
- `g.id.net.email`
- `g.id.net.msisdn`

All values must be given under the key `identifier.value` of the request object.

Example:

```json
{
    "session": "RandomSessionID",
    "type": "g.auth.password",
    "identifier": {
        "type": "g.id.email"
        "value": "john@example.org"
    }
}
```

##### Fallback

The namespace `/auth/{authType}/fallback/{fallbackType}` is available to use under the base namespace of each API.

Example for use on the Identity API for a simple password authentication in a browser:

```
GET /identity/client/v0/auth/g.auth.password/fallback/g.fallback.web?session=RandomSessionID&redirectUrl=http%3A%2F%2Flocalhost%3A65432%2Fauth%3Fsession%3DRandomSessionID
```



##### Fallback Types

###### Web Browser

**Type:** `g.fallback.web`
**Usage:** To be displayed in a browser.

The following query parameters are available:

| **Key**       | **Required** | **Purpose**                                            |
| ------------- | ------------ | ------------------------------------------------------ |
| `session`     | **Yes**      | The session ID to complete the stage for               |
| `redirectUrl` | **Yes**      | The URL to redirect the client used in the fallback to |

#### Authorisation

Clients and servers can adapt several requirement levels for exchange of such credentials.
Four type of credential checks are available:

- **MUST**: Using a valid token is mandatory; No choice.
  - Clients **MUST** send credentials. If no credentials is available, the request **MUST NOT** be made.
  - Servers **MUST NOT** let clients use the endpoint if not authorised.
- **SHOULD**: Using a valid token is expected; Don't expect it to work without, but may work in specific cases.
  - Clients **SHOULD** send credentials.
  - Servers **SHOULD** deny the request unless they want to grant anonymous access to specific resources, in which case they **MUST** be able to restrict to the specific set of clients that the special behaviour is targeting.
- **MAY**: Using a valid token is not needed but is encouraged; It's possible the server will return an enhanced response if the user is authenticated.
  - Clients **SHOULD** send credentials.
  - Servers **MUST** accept endpoint usage with or without credentials. Servers **SHOULD** enhance the request as much as possible if the client is identified and **SHOULD** redact as much info as possible if credentials are not available.
- **MUST NOT**: Never send credentials. Ever.
  - Clients **MUST NOT** send credentials.
  - Servers **MUST** fast-fail the request if credentials are provided.

> **TODO**: be clear of the distinction between "credentials (not) available", "client authenticated", "client authorised", etc.

### Errors

*TBC*