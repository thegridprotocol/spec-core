# Grid Protocol - Architecture

## Concepts

### Channels

Persistent data is exchanged over a fundamental structure called `channel`.

## Data

[JSON](https://en.wikipedia.org/wiki/JSON) is the baseline and default data format to be supported by all implementations. MIME type is `application/json`. 

[UTF-8](https://en.wikipedia.org/wiki/UTF-8) is the baseline and default data encoding to be supported by all implementations.

When exchanging data between implementations, if there is a possiblity to specify the Mime Type, Implementations `SHOULD NOT` append a charset like `; charset=utf-8` if the encoding is `UTF-8`.

>  **Rationale**: Keep it straight-forward for implementations.

## Network

[HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol) is the baseline and default transport to be supported by all implementations. Implementations `SHOULD` use [HTTP/2](https://en.wikipedia.org/wiki/HTTP/2) ([RFC 7540](https://tools.ietf.org/html/rfc7540)) by default and only fallback to HTTP/1.1 ([RFC 7230](https://tools.ietf.org/html/rfc7230)) if HTTP/2 is not available. Any prior version SHOULD NOT be used.

>  **Rationale**: Promote `HTTP/2` now that it is a stable RFC to promote acceptance. `HTTP/3` is too young and might burden implementations unnecessarly at this stage of the protocol, but implementations are encouraged to try and report on it.

[TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security) 1.2 is the baseline and default security layer for HTTP. Implementations that provide a mean to connect over any kind of public network `MUST` support it and use it as default.

> **Rationale**: Enforce the privacy value of the project.

## Identifiers

Identifiers will be split into two categories:

- `ID`: Opaque low-level Identifier used by implementations to uniquely identify and route data.
- `Address`: High-level Identifier to be created and used by users.

### IDs

An `ID` is defined as a compound string made of a single character in first position, called `sigil` followed by a unique network identifier.

Except for the `sigil` defining the ID type, characters `MUST` be from the [Base64](https://tools.ietf.org/html/rfc4648) encoding without padding and use the [URL-safe variant](https://tools.ietf.org/html/rfc4648#section-5).

> **Rationale**:
>
> - Without padding: keep IDs as small as possible, as they will repeat in each message
> - URL-safe variant: avoid any mess in URLs with things like `/` and avoid implementations bugs that still treat `/` and other similar characters as special characters even after decoding.

Example:

```
@fdsD-5_43
```

- `@` is the `sigil`
- `fdsD-5_43` is the unique network identifier

#### Ecosystem bootstrap

To bootstrap the ecosystem, IDs `MUST`be built like this:

1. Pick an arbitrary unique identifier for the configured domain
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

For this version of the spec, Implementations are expected to decode IDs the same way to be able to route messages and initiate connections.

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

