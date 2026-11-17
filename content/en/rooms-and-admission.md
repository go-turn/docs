---
title: "Rooms and admission"
weight: 60
---

# Rooms and admission {#rooms-and-admission}

A room is how two peers meet with one link: a host opens it, shares the link, and
the guest's answer comes back on its own.

## Knocking, not entering {#knocking}

A leaked room link lets a stranger **knock**, not enter. The room ID is 128 random
bits, and a guest who joins with it is **pending**:

- a pending guest only receives: it cannot send and cannot get credentials;
- it counts against a separate pending cap, and a pending TTL;
- the host sees `sys.knock` (`from` = the guest, `body` = its display name, at most
  64 characters, no control characters) and decides.

Only admitted guests count against `--max-guests` and produce `sys.join`.

## Deciding {#deciding}

| call | by | effect |
|---|---|---|
| `Admit{room,peer}` | the host | admits a pending guest |
| `Deny{room,peer}` | the host | refuses a pending guest, or removes an admitted one: its token is dead |
| `RoomCredentials{room}` | an admitted peer | `IceServers` expiring with the room, charged to the host |
| `CloseRoom{room}` | the host | ends the room |

## System events {#system-events}

Delivered on the room's one stream (`Room`), events down, messages up:

| event | |
|---|---|
| `sys.knock` | a guest is pending |
| `sys.admitted` | |
| `sys.denied` | |
| `sys.join` | an admitted guest joined |
| `sys.leave` | |
| `sys.closed` | the room ended |
| `sys.error` | a message the other peer could not take; the stream stays open |

Ending the stream is leaving. A second stream for one peer is refused with
`ALREADY_EXISTS`.

## What breaks the protocol {#protocol-breaks}

A message that is too large, a spoofed `from`, a bad kind, one sent by a pending
guest, or one from guest to guest ends the sender's stream with a status
(`RESOURCE_EXHAUSTED`, `PERMISSION_DENIED`, `INVALID_ARGUMENT`); see
[The gRPC API]({{< relref "grpc-api.md#status-codes" >}}).
