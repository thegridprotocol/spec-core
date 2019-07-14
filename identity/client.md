# The Grid - Identity - Client to Server API

**WARNING**: This document is Work in Progress. Attempts at implementations are encouraged and will be supported. Reviews and comments are also welcome.

## Requirements

### RFCs

The following RFCs are prerequisites to this document:

- [2119](https://tools.ietf.org/html/rfc2119)  - *Key words for use in RFCs to Indicate Requirement Levels*
- [2606](https://tools.ietf.org/html/rfc2606) - *Reserved Top Level DNS Names*
- [8174](https://tools.ietf.org/html/rfc8174) - *Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words*

### Grid

The following spec documents are prerequisites to this document:

- [Fundamental Concepts](../concepts.md) 

## Overview

This document describes the API that can be used in the scope of Identity exchanges between a client and a server.

## Baseline

All the endpoints **MUST** be prefixed with `/identity/client`.

### Authorization

*TBC*

### Errors

*TBC*

### Server discovery

Via `.well-known` at `/.well-known/grid/identity/client`

With format:

```json
{
    "v": "0",
    "api": [
        {
            "base_url": "https://example.org/_grid" # Top domain style
        }
        {
            "base_url": "https://grid.example.org" # Sub domain style
        }
    ]
}
```

Top level keys define the entry.`

`api` is an array of objects containing the following key(s):

| Key        | Purpose                                             |
| ---------- | --------------------------------------------------- |
| `base_url` | Mandatory. Base URL to the API without trailing `/` |

Clients need to try to use them in order until one server reply. Caching of data is controlled via regular HTTP headers, but should not be cached between client restarts.

## Status

### `GET /versions`

Credentials: **MAY**

Get the support version of the API from the server.

#### Response

Status: `200`

Body:

```json
{    
    "api": [
        "v0",
        "arbitrary.version.v34032"
    ],
    "server": {
        "name": "Awesome-Grid-Data-Server-Implementation",
        "version": "v0.0.1-WE_ARE_SUPER_ALPHA"
    }
}
```

### `GET /v0/`

**Credentials**: **MAY**

Check if the `v0` API is available. **MAY** act as a ping to the server.

#### Response

Status: `200`

Body:

```json
{}
```

## Auth

### `/v0/login`

Credentials: **MUST NOT**

Perform login to get an access token.

#### Request

See [Concepts: API Authentication](../concepts.md#authentication).

#### Response

If success:

Status: `200`

Body:

```json
{
    "token": "abclfkskfrokfo43423"
}
```



### `GET /v0/logout`

Credentials: **MUST**

Perform a logout for this session.

#### Response

If success:

Status: `200`
Body:

```json
{}
```

## Identities

### `GET /v0/identifier/`

List all available identifiers.

#### Request

*No options.*

#### Response

##### Success

**Status:** 200
Body:

```json
{
    "items": [
        "@userID-James",
        "@userID-Bond"
    ]
}
```


### `GET /v0/identifier/{id}`

Get information about a specific identifier.

### `GET /v0/identifier/{id}/server/`

List data servers linked to the identifier.

### `POST /v0/do/lookup/user/threepid`

Lookup the ID of a user using a 3PID.

#### Request

Body:

```json
{
    "identifier": {
        "type": "g.id.net.grid.alias",
        "value": "@john@example.org",
    }
}
```

#### Response

##### Success

**Status:** 200
**Body:**

```json
{
    "id": ":VGhpc0lzTWUsSm9obiE"
}
```

##### Failure

Not found: `404`  with a standard error body using `errcode` value of `G_NOT_FOUND`.