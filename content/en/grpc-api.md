---
title: "The gRPC API"
weight: 50
---

# The gRPC API {#the-grpc-api}

The API is gRPC, [`proto/turn/v1/turnd.proto`](https://github.com/go-turn/turn/blob/main/proto/turn/v1/turnd.proto)
(service `turn.v1.Turnd`). The generated code is committed, and CI regenerates it
and fails on a diff.

## Transport {#transport}

A plain `grpc.Server` is served over a WebSocket at `/grpc`
([grpc-transports/websocket](https://github.com/grpc-transports/websocket)), which
is what a browser can speak, including the bidirectional stream.

The WebSocket upgrade enforces the exact origin allowlist (`--allowed-origin`;
scheme and port included). A client with no `Origin` header is a native client.

## Tokens {#tokens}

A token travels as `authorization: Bearer …` metadata, never in a URL.

## HTTP, for browser navigations {#http}

Only browser navigations stay HTTP:

| HTTP | |
|---|---|
| `GET /login?origin=<allowed origin>` | starts the OIDC login (code + PKCE) in a popup; an origin not in `--allowed-origin` is a 400, never a redirect |
| `GET /callback` | page that `postMessage`s `{type:"turnd-session",token,expires}` (or `{type:"turnd-error",error}`) to `window.opener`, target origin = the allowlisted one |
| `GET /healthz` | `ok` |

## RPCs {#rpcs}

| RPC | token | returns |
|---|---|---|
| `Credentials` | session, or issuer access token | `IceServers{ice_servers[{urls,username,credential}], ttl}` |
| `CreateRoom` | session or access token | `{room, peer:"g1", token, expires, max_guests}`: the creator's peer ID is ordinary (breaking since v0.7: it was `"host"`); the room ID is 128 random bits; the request may carry `max_guests` |
| `JoinRoom{room,name}` | none: the room ID | `{peer, host, token, expires, pending:true}`; the guest is **pending** |
| `Admit{room,peer}` / `Deny{room,peer}` | host token | admit a pending guest / refuse it (or remove an admitted one): its token is dead |
| `RoomCredentials{room}` | an **admitted** peer's token | `IceServers` expiring with the room, charged to the host |
| `CloseRoom{room}` | host token | ends the room |
| `Room(stream Envelope) returns (stream Envelope)` | peer token + `x-turn-room` metadata | the room's one stream: events down, messages up; ending it is leaving |

What the room RPCs mean is in [Rooms and admission]({{< relref "rooms-and-admission.md" >}}).

## Status codes {#status-codes}

A message that breaks the protocol (too large, a spoofed `from`, a bad kind, a
pending sender, guest to guest) ends the sender's stream with a status:
`RESOURCE_EXHAUSTED`, `PERMISSION_DENIED`, `INVALID_ARGUMENT`. Other refusals:

| status | when |
|---|---|
| `UNAUTHENTICATED` | no or dead token |
| `NOT_FOUND` | |
| `ALREADY_EXISTS` | a second stream for one peer |
| `FAILED_PRECONDITION` | already admitted |
| `UNAVAILABLE` | server full |

## The credential scheme {#the-credential-scheme}

[draft-uberti-behave-turn-rest-00](https://datatracker.ietf.org/doc/html/draft-uberti-behave-turn-rest-00).

## The Go client {#the-go-client}

For the browser and native: `github.com/go-turn/turn/rendezvous/client`.
`Create`, `Join(name)` (blocks until admitted, `ErrDenied` otherwise), `Admit`,
`Deny`, `Send`, `Recv`, `Credentials`, `Close`.

It is a real grpc-go client dialed over the browser's WebSocket through
`syscall/js`. It adds about 16 MB to a wasm binary (3.5 MB more gzipped), almost
all of it grpc. go-crdt/collab's js/wasm build does not link grpc, so a page that
does not otherwise ship it should load this client in a separate wasm, fetched
when a collaboration starts.
