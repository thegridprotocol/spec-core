# The Grid - Data - Server to Server APIs

The following RFCs are prerequisites to this document:

- [2119](https://tools.ietf.org/html/rfc2119)  - *Key words for use in RFCs to Indicate Requirement Levels*
- [8174](https://tools.ietf.org/html/rfc8174) - *Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words*
- [2606](https://tools.ietf.org/html/rfc2606) - *Reserved Top Level DNS Names*

**WARNING**: This document is Work in Progress. It assumes the reader has prior good knowledge of the Matrix S2S API on which this document is based, until all sections are filled. Some info is also left generic/incomplete and one should assume the Matrix endpoints are just ported as-is except for the API Baseline directives. In doubt, follow the Matrix spirit.

**WARNING**: This API is currently very volatile and considered bleeding edge. Reviews and comments are welcome and encouraged as long as they keep in mind the purposeful lack of information on some topics/endpoints in the short-term.

## Overview

This document describes the API that can be used for data exchanges between two data servers.

## Baseline

All the endpoints **MUST** be prefixed with `/_grid/data/server`.

### Authorization

> TBC, via headers like Matrix

### Errors

> TBC

### Server Discovery

Via `.well-known` at `/.well-known/grid/server`

With format:

```json
{
    "v": "0",
    "g.data": [
        {
            "base_url": "https://example.org" # Top domain style
        }
        {
            "base_url": "https://grid.example.org/data" # Sub domain style
        }
    ]
}
```

Top level keys define the entry.

Official keys start with `g.`

Custom keys must use Java package naming convention

`g.data` is an array of objects containing the following key(s):

| Key        | Purpose                                             |
| ---------- | --------------------------------------------------- |
| `base_url` | Mandatory. Base URL to the API without trailing `/` |

Servers need to try to use them in order until one server reply. Caching of data is controlled via regular HTTP headers, but should not be cached between server restarts.

## Synchronization

### `POST /v0/do/push`

Send events to another server.

The request **MUST** contain at least one event. The request **SHOULD** be limited to a reasonable amount of events, but no hard upper limit is set. The recommendation is to limit requests to 50 events.

The receiving server **CAN** fail requests based on the `Content-Length` header which **MUST** be set by the sending server, as long as they are bigger than a single event max size plus the request body structure overhead.

> **TODO:** Calculate the actual min size for accepted failures

#### Request

```json
{
    "events": [
        {
            "v": "0",
            "type": "g.example",
            "content": {
                "This is": "just an example"
            }
        },
        {
            "v": "0",
            "type": "g.example",
            "content": {
                "This is": "just another example"
            }
        }
    ]
}
```

#### Response

##### Success

**Status:** `200`

##### Failure

The following failures are possible:

| Status | Code                        | Description                                                  |
| ------ | --------------------------- | ------------------------------------------------------------ |
| `400`  | `G_CONTENT_LENGTH_EXCEEDED` | The server refused to process the request, being too big. The sender should break up the request and try again. |
| `411`  | `G_MISSING_HEADER`          | The `Content-Length` header is missing or has no numeric value |

### `GET /v0/do/fetch`

Used by a server on another to download any pending push that might have not be successful. This is somewhat similar to the client `/sync` endpoint, except that this must only be called occasionally and if push was not available, like after being offline for a certain amount of time.

> **TODO**: Complete with request/response. This would be a mirror of `/push`.

## Approvals

Certain actions require that the server being the target, or being authoritative for a user that it is hosting, approves it.

The following event types require approval:

- `g.c.member` if the key `content.action` has a value of `invite`

### `POST /v0/do/approve/invite`

Approve an invite event.

#### Request

```json
{
    "object": {
        "v": "0",
        "origin": ":originServer",
        "channel": "#chanID",
        "sender": "@senderID",
        "scope": "@inviteeID",
        "type": "g.c.member",
        "content": {
            "action": "invite"
        },
        "signatures": {
            ":originServer": "SomeBase64Here"
        }
    },
    "context": {
        "state": [
            {
                "A first": "state event"
            },
            {
                "A second": "state event"
            }
        ]
    }
}
```

| Key             | Type                    | Required | Purpose                                                                              |
| --------------- | ----------------------- | -------- | ------------------------------------------------------------------------------------ |
| `object`        | Channel Event           | **Yes**  | The event to be signed by the server.                                                |
| `context.state` | Array of Channel Events | **Yes**  | The state of the channel at the time the invite was requested. This cannot be empty. |

#### Response

##### Success

**Status:** `200`
**Body:**

```json
{
    "data": {
        "v": "0",
        "origin": ":originServer",
        "channel": "#chanID",
        "sender": "@senderID",
        "scope": "@inviteeID",
        "type": "g.c.member",
        "content": {
            "action": "invite"
        },
        "signatures": {
            ":originServer": "SomeBase64Here",
            ":remoteServer": "SomeOtherBase64Here"
        }
    }
}
```

##### Failure

The following failures are possible:

| Status | Code        | Description                                                                                                                           |
| ------ | ----------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `403`  | `G_REFUSED` | The server refuses to counter-sign the event. The server **SHOULD** give a meaningful error message so it can be relayed to the user. |

