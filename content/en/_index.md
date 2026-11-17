---
title: "Home"
weight: 10
---

# go-turn {#go-turn}

**STUN and TURN in pure Go**, for WebRTC peers that cannot reach each other
directly — a symmetric NAT, a full-tunnel VPN, a network that only lets HTTPS
out. `CGO_ENABLED=0`.

One daemon, `turnd`, serves on one host:

| | |
|---|---|
| **STUN** | Binding (RFC 8489): a peer learns the address the Internet sees |
| **TURN** | a relayed UDP address (RFC 8656), reached over UDP, TCP, or TLS on 443 (`turns:`) |
| **credentials** | short-lived TURN credentials (HMAC, the TURN REST API scheme), minted only after an OpenID Connect login through [go-authn](https://github.com/go-authn) |
| **rendezvous** | a room a host opens and shares as one link; the guest's answer comes back on its own |

It is the relay the [go-tex playground](https://go-tex.github.io/playground/)
uses for live collaboration over [go-crdt/collab](https://github.com/go-crdt/collab).

## Where to go {#where-to-go}

| You want to… | Read |
|---|---|
| put `turnd` on a host | [Deploying]({{< relref "deploying.md" >}}) |
| serve TURN over TLS on 443 beside an HTTPS site | [TURN over TLS on 443]({{< relref "tls-on-443.md" >}}) |
| prove a deployed relay works | [Checking a deployment]({{< relref "checking-a-deployment.md" >}}) |
| call the API from a browser or a Go program | [The gRPC API]({{< relref "grpc-api.md" >}}) |
| understand what a room link allows | [Rooms and admission]({{< relref "rooms-and-admission.md" >}}) |
| know what the relay refuses to do | [Security policy]({{< relref "security.md" >}}) |
| use the packages directly | [Packages]({{< relref "packages.md" >}}) |

## Status {#status}

The relay, the credentials, the login and the rendezvous work and are tested.
The design is [turn#1](https://github.com/go-turn/turn/issues/1); what is still
open is in the [tracker](https://github.com/go-turn/turn/issues).
