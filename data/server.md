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

All the endpoints below **MUST** be prefixed with `/data/server`.

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
            "base_url": "https://example.org/_grid" # Top domain style
        }
        {
            "base_url": "https://grid.example.org" # Sub domain style
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

## Status

### `GET /version`

Get the version info about a server. This endpoint is also consider the "ping" endpoint and servers **MUST** only use this one to check if a remote server is alive. This endpoint **MUST** not be cached by infrastructure components.

> Rationale: give a clear endpoint to can be used to know if a server is alive which can be hard-coded in implementation. The main planned usage of this endpoint outside of version checking would be checking the availability of a server before accepting a join, regardless of cached signed keys. At this point we are not sure if it will make a real difference, so we will try during beta and see.

#### Response

##### Success

**Status:** 200

**Body:**

```json
{
  "api": [
    "v0"
  ],
  "server": {
      "name": "SomeServerImplementation"
      "version": "v1.33.7"
  }
}
```

**Root object**

| Key      | Type     | Required | Purpose                                       |
| -------- | -------- | -------- | --------------------------------------------- |
| `api`    | array    | **Yes**  | List of supported API versions by the server. |
| `server` | `Server` | No       | Implementation details.                       |

**`Server` object**

| Key       | Type   | Required | Purpose                        |
| --------- | ------ | -------- | ------------------------------ |
| `name`    | string | **Yes**  | Name of the implementation.    |
| `version` | string | No       | Version of the implementation. |



## Data access

The following endpoints allow fundamental data access to the fundamental structure of the protocol.

### `GET /v0/channels/{channelId}/events/{eventId}`

Get a specific event from a channel.

#### Request

Path variables:

| Name        | Required | Purpose                                           |
| ----------- | -------- | ------------------------------------------------- |
| `channelId` | **Yes**  | The channel ID in which to find locate the event. |
| `eventId`   | **Yes**  | The event ID of the event.                        |

#### Response

##### Success

**Status:** 200
**Body:**

```json
{
    "event": {
        "Some": "Channel event"
    }
}
```

| Key             | Type                    | Required | Purpose                                                                              |
| --------------- | ----------------------- | -------- | ------------------------------------------------------------------------------------ |
| `event`    | Channel Event           | **Yes**  | The requested channel event.                    |


##### Failure

The following failures are possible:

| Status | Code          | Description                                                  |
| ------ | ------------- | ------------------------------------------------------------ |
| `403`  | `G_FORBIDDEN` | The requesting server is not allowed to see the event. |
| `404`  | `G_NOT_FOUND` | The channel or the event are not known to the server. |

## Synchronisation

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

Certain actions require approval from the server being the target - or representing a user - of an event, or involved in a process, like joining a public room without prior interaction.

The approval process allows a first layer of control and validation on both ends, and is especially designed to fight spam or unwanted solicitations.

Approvals are created by having the targeted servers counter-sign an event after evaluating the event itself and the context surrounding the event.

### `POST /v0/do/approve/invite`

Approve an invite event. More formally, This endpoint is called on Server B by Server A when attempting to invite User C from Server B.

The qualifying event is of type `g.c.s.member` when the key `content.action` has the value `invite`.

#### Request

Example body:
```json
{
    "event": {
        "v": "0",
        "origin": ":originServer",
        "channel": "#chanID",
        "sender": "@senderID",
        "scope": "@inviteeID",
        "type": "g.c.s.member",
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
| `event`        | Channel Event           | **Yes**  | The event to be signed by the server.                                                |
| `context.state` | Array of Channel Events | **Yes**  | The state of the channel at the time the invite was requested. This **MUST NOT** be empty. |

#### Response

##### Success

**Status:** `200`
**Body:**

```json
{
    "event": {
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

| Key             | Type                    | Required | Purpose                                                                              |
| --------------- | ----------------------- | -------- | ------------------------------------------------------------------------------------ |
| `event`    | Channel Event           | **Yes**  | The event approved and counter-signed by the server.                    |

##### Failure

The following failures are possible:

| Status | Code        | Description                                                                                                                           |
| ------ | ----------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `403`  | `G_REFUSED` | The server refuses to counter-sign the event. The server **SHOULD** give a meaningful error message so it can be relayed to the client/user. |

### `POST /v0/do/approve/join`

Approve a join event. More formally, This endpoint is called on Server B by Server A when a user from Server A wants to join a public room without prior invitation.

The qualifying event is of type `g.c.s.member` when the key `content.action` has the value `join`, when the state has an event `g.c.s.joining` when the key `content.rule` has the value `public`.

#### Request

Example body:
```json
{
    "event": {
        "v": "0",
        "origin": ":originServer",
        "channel": "#chanID",
        "sender": "@senderID",
        "scope": "@senderID",
        "type": "g.c.s.member",
        "content": {
            "action": "join"
        },
        "signatures": {
            ":originServer": "SomeBase64Here"
        }
    }
}
```

| Key             | Type                    | Required | Purpose                                                                              |
| --------------- | ----------------------- | -------- | ------------------------------------------------------------------------------------ |
| `event`        | Channel Event           | **Yes**  | The event to be approved by the server.                                     |

#### Response

##### Success

**Status:** `200`
**Body:**

```json
{
    "event": {
        "v": "0",
        "origin": ":originServer",
        "channel": "#chanID",
        "sender": "@senderID",
        "scope": "@inviteeID",
        "type": "g.c.s.member",
        "content": {
            "action": "join"
        },
        "signatures": {
            ":originServer": "SomeBase64Here",
            ":remoteServer": "SomeOtherBase64Here"
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
| `data`        | Channel Event           | **Yes**  | The event to be signed by the server.                                                |
| `context.state` | Array of Channel Events | **Yes**  | The state of the channel at the time the join was approved. This **MUST NOT** be empty. |

##### Failure

The following failures are possible:

| Status | Code        | Description                                                                                                                           |
| ------ | ----------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `403`  | `G_REFUSED` | The server refuses to counter-sign the event. The server **SHOULD** give a meaningful error message so it can be relayed to the client/user. |

## Lookups

### `POST /v0/do/lookup/channel/alias`

Get the Channel ID and resident data server(s) for the channel.

#### Request

**Body**:

```json
{
    "alias": "#channelAlias"
}
```
| Key             | Type                    | Required | Purpose                                                                              |
| --------------- | ----------------------- | -------- | ------------------------------------------------------------------------------------ |
| `alias`    | string           | **Yes**  | The Channel Alias being looked up.                    |

#### Response

##### Success

**Status:** 200
**Body:**

```json
{
    "id": "#channelID",
    "servers": [
        ":serverID-A",
        ":serverID-B"
    ]
}
```

> **TODO:** This endpoint does not include any way to resolve Server IDs to routable addresses. While the current spec has the routing info builtin the ID, this will only be true for v0. We need to adapt this endpoint by then.

##### Failure

The following failures are possible:

| Status | Code          | Description                                                  |
| ------ | ------------- | ------------------------------------------------------------ |
| `404`  | `G_NOT_FOUND` | The channel alias is unknown to the server. The server **SHOULD** use this status code to reply to blacklisted/ban servers, instead of a `403` to ensure blacklists/bans are not leaked. |
