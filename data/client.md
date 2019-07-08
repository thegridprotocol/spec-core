# The Grid - Data - Client to Server APIs

**WARNING**: This document is Work in Progress. It assumes the reader has prior good knowledge of the Matrix C2S API on which this document is based, until all sections are filled. Some info is also left generic/incomplete and one should assume the Matrix endpoints are just ported as-is except for the API Baseline directives. In doubt, follow the Matrix spirit.

**WARNING**: This API is currently very volatile and considered bleeding edge. Reviews and comments are welcome and encouraged as long as they keep in mind the purposeful lack of information on some topics/endpoints in the short-term.

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

This document describes the API that can be used in the scope of data exchanges between a client and a server.

## Baseline

All the endpoints **MUST** be prefixed with `/_grid/data/client`.

### Server discovery

Via `.well-known` at `/.well-known/grid/client`

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

## Authorisation

Access to endpoints **MAY** be restricted by the use of credentials from the client to the server.

*TBC*

## Synchronization

### `GET /v0/sync`

*TBC - Assume like Matrix with naming changes*

## Channels

### `POST /v0/do/create/channel`

Credentials: **DO IT**

Create a new channel and return its ID.

#### Response

##### Success

Status: 200

Body:

```json
{
    "id": "#abcdefgABCDEF"
}
```

##### Failure

Status 401 is credentials are mandatory

> TBC with body

Status 403 if creating a channel is not allowed

>  TBC with body

### `GET /v0/channels/{channelIdOrAddress}/timelines/events/`

Credentials: **SHINNIES**

Get the most recent N events in **Timeline order** in a simple client format. This can be used for peeking or to see the timeline without credentials.

> **Note:** Timeline order is defined in the Base specification.

> **TODO**: Define paging: Assume Matrix-style for now.

#### Response

If success:
Status: `200`
Body:

```json
{
    "events": [
        {
            "v": "0.0",
            "type:" "g.c.create",
            "...": "..."
        },
        {
            "v": "0.0",
            "type": "g.c.member",
            "sender": "@senderUserID",
            "scope": "@invteeUserID"
            "content": {
                "membership": "join"
            }
        },
        {
            "v": "0.0",
            "type": "g.c.message",
            "...": "...",
            "content": {
                "body": {
                    "text/plain": "Blah in plain text",
                    "text/html", "<h1>Blah</h1>In HTML super quality"
                }
            }
        }
    ]
}
```

> **NOTE**: Just an example, non-normative

### `POST /v0/channels/{channelIdOrAddress}/timelines/events/{txnId}`

Credentials: **DO IT**

Send an event to a room in timeline mode (do DAG linking on server-side).

The `{txnId}` path parameter is used to deduplicate requests in case of network issues.

#### Request

Body:

```json
{
    "type": "g.c.message",
    "content": {
        "body": {
            "text/plain": "Just a simple text message",
            "text/html": "I am a fancy <b>HTML<b> view of a simple text message"
        }
    }
}
```

State event can be sent by setting a `scope` key.

#### Response

If success:

Status: `200`
Body:

```json
{
    "id": "$aldksfo34324543"
}
```

### `GET /v0/channels/{channelIdOrAddress}/timelines/states/`

Get the most recent N state events in **Timeline order** in a simple client format.

### `GET /v0/channels/{channelIdOrAddress}/dags/events/`

Get the event DAG latest info (like extremities)

### `PUT /v0/channels/{channelIdOrAddress}/dags/events/`

Send a new event in full format with all the required fields. The server **SHOULD NOT** change anything on the event.

### `GET /v0/channels/{channelIdOrAddress}/dags/states/`

Get the state DAG latest info (like extremities).

### `GET /v0/channels/{channelIdOrAddress}/do/`

This and any `/do/` paths are convenience endpoint for clients that map to predefined events. The same action can be performed by sending plain events to the channel.

> **Note:** These endpoints are meant to be used by "dummy" clients who cannot necessarily perform all required checks and are meant to make it easy for them by having the server perform any prerequisite action(s) for them, if such prerequisite action(s) is possible.
>
> **Note:** This is just an idea and at this point to be discussed. They are not mandatory in PoC implementations.

This specific endpoint can be used to retrieve the list of possible actions.

 #### Response

Body:

```json
{
    "do": [
        {
            "path": "join",
            "description": "Join the room"
        },
        {
            "path": "kick",
            "description": "Kick a joined member from the room"
        },
        "..."
    ]
}
```

   ### `PUT /v0/channels/{channelIdOrAddress}/do/invite`

### `PUT /v0/channels/{channelIdOrAddress}/do/join`

### `PUT /v0/channels/{channelIdOrAddress}/do/subscribe`

> **NOTE:** Unsubscribing is done using `/leave`.

### `PUT /v0/channels/{channelIdOrAddress}/do/leave`

> **NOTE:** Can also be used for kicks, as kick is performing `leave` on another user.

### `PUT /v0/channels/{channelIdOrAddress}/do/ban`

> **NOTE:** Unban is done using `/leave`.
