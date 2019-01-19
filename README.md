# The Grid Protocol - Architecture

The following RFCs are prerequisites to this document:

- [2119](https://tools.ietf.org/html/rfc2119)  - *Key words for use in RFCs to Indicate Requirement Levels*
- [2606](https://tools.ietf.org/html/rfc2606) - *Reserved Top Level DNS Names*
- [8174](https://tools.ietf.org/html/rfc8174) - *Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words*

## Concepts

### Channels

Persistent data is exchanged over a fundamental structure called `channel`.

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
>  `HTTP/3` is too young and might burden implementations too much at this stage of the protocol if used as the default transport. Implementations are still encouraged to try and report on it in the *Exchanges* section's intro paragraph.
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

- `ID`: Opaque low-level Identifier used by implementations to uniquely identify and route data.
- `Address`: High-level Identifier to be created and used by users.

### IDs

An `ID` is defined as a compound string made of a single character in first position, called `sigil` followed by a globally unique network identifier.

An `ID` **MUST** be treated as an opaque, case-sensitive string and used as-is.

> **Open Question**: *Opaque* is not defined in a technical manner, and may be unclear and/or missleading.
> **Is there a better word that states that implementations must not try to make sense of it, unless specifically stated in this doc?**


> **Rationale**: an `ID` is a low-level identifier and is therefore not meant to be used by humans/users directly. Given the privacy goal of the protocol, it is also important to split the concepts of `ID` not inherently tied to an identifiable information and `Address` which is specifically created by a user/system so an entity can specifically be identified.
>
> Making an `ID` opaque and case-sensitive allows for them to be treated as bytes without putting any kind of restriction for the future, while their companion Identifier, `Addresses` deal with user-specific needs.

Except for the `sigil` defining the ID type, characters **MUST** be from the [Base64](https://tools.ietf.org/html/rfc4648) encoding without padding and use the [URL-safe variant](https://tools.ietf.org/html/rfc4648#section-5).

> **Rationale**:
>
> Without padding*: keep IDs as small as possible, as they will repeat in each event.*
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

To bootstrap the ecosystem, IDs **MUST**be built like this:

1. Pick an arbitrary unique identifier for the object, unique within the configured realm.
2. Append `@`
3. Append the domain
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

Sigil: `:`

#### Users

Sigil: `@`

#### Channels

Sigil: `#`

#### Events

Sigil: `$`

### Addresses

An `address` is defined as a compound string made of three elements, in order:

- A `sigil` to define the type of `address`
- The character `@` 
- A server address

#### Users

Sigil: `@`

Example:

```
@john@example.org
```

#### Channel

Sigil: `#`

Example:

```
#grid@example.org
```

