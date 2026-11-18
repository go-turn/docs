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
| `HandOver{room,to}` | the host | gives the host role to an admitted, connected guest |
| `AppointCoHost{room,peer}` / `RevokeCoHost{room,peer}` | the host only | an admitted guest may admit, deny and close beside the host; revoking is immediate |
| `Reclaim{room}` | a session token of the host's subject | the host role back, or an admitted guest's seat after a handover |

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
| `sys.cohost-added`, `sys.cohost-removed` | to everyone: body `{"peer":"<peer>"}` |
| `sys.host-changed` | to everyone: the host role moved, body `{"from":"<peer>","to":"<peer>"}` |

Ending the stream is leaving. A second stream for one peer is refused with
`ALREADY_EXISTS`.

## A room outlives its host {#host-handover}

Collaboration is peer to peer, so a room must not die with the browser that opened
it. When the host's stream ends, the room lives on for `--host-grace` (default
`2m`; a negative value restores the old behaviour: the host leaving closes the room).

**Within the grace** the host, back as the same person, keeps the role and nothing
changes for the others: `Reclaim` with a session token of the host's subject (a new
host token, the old one dead), or a new `Room` stream with its own token. Messages
sent to `"host"` meanwhile are queued for it (up to `--max-queue`). Only a message
written to the old connection before the server noticed it was dead can be lost.

**Past the grace**, or when the host calls `HandOver{room,to}` first, the
earliest-appointed co-host still connected, else the **earliest-admitted guest still
connected**, becomes the host. There is no
version-vector election: a rejoin syncs both ways, so the collab layer does not need
it. A pending, denied or disconnected guest is never promoted; if nobody qualifies,
the room closes. A guest cannot promote itself, and a non-host cannot `HandOver`
(`PERMISSION_DENIED`). There is exactly one host at any moment.

**Co-hosts.** The host, and only the host, appoints an admitted guest
(`AppointCoHost`) and revokes it (`RevokeCoHost`); a pending or denied peer cannot be
appointed. A co-host may admit, deny (not the host or another co-host) and close the
room, and its room credentials are charged to its own subject if it presented a
session, else to the host's, never past the room's TTL. A co-host cannot appoint,
revoke or hand over: there is exactly one active host. Everyone receives
`sys.cohost-added` / `sys.cohost-removed` (`{"peer":"<id>"}`). Knocks, joins and
leaves go to the host and to every co-host, so a waiting guest's expiry
(`MaxPending`, `PendingTTL`) now has several people to answer it. On a host loss the
earliest-appointed co-host still connected is promoted before any guest.

What changes:

- The promoted peer keeps its **peer ID and token**, so the site identity a collab
  layer derives from the ID is stable. It gains `Admit`, `Deny`, `HandOver`,
  `CloseRoom`. The old host's token is dead.
- Everyone, the old host included, receives `sys.host-changed {from,to}`. The new
  host receives `sys.knock` again for each pending guest, `sys.join` for each
  admitted one, then the messages queued for the old host, in order. `"host"` as a
  guest's destination always means the current host.
- `RoomCredentials` are charged to the current host's person if it said who it was
  when it knocked (`x-turn-session` metadata, `client.WithSession`), else to the
  person who opened the room. The room's TTL is **never** extended.
- The old host returning after a promotion calls `Reclaim`: same peer ID, an
  **admitted** guest of the new host (no knock; the new host sees `sys.join`).

On `sys.host-changed`, the collab layer does the following: the new host starts its
`collab.Server` seeded from its own replica; the others rejoin it through the
rendezvous with their version vector, and a guest that is ahead pushes what it
holds. In peer-to-peer mode nothing may purge history.

In the Go client (`rendezvous/client`): `Room.IsHost()`, `Room.CurrentHost()`,
`Event.HostChange()`, `Room.HandOver`, `client.Reclaim`, and the `WithSession` join
option.

## What breaks the protocol {#protocol-breaks}

A message that is too large, a spoofed `from`, a bad kind, one sent by a pending
guest, or one from guest to guest ends the sender's stream with a status
(`RESOURCE_EXHAUSTED`, `PERMISSION_DENIED`, `INVALID_ARGUMENT`); see
[The gRPC API]({{< relref "grpc-api.md#status-codes" >}}).
