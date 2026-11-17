---
title: "Security policy"
weight: 70
---

# Security policy {#security-policy}

What `turnd` refuses to do, as stated by the project.

## The relay does not reach inward {#relay-address-policy}

By default the server never relays to or from:

- loopback, `0.0.0.0/8`, RFC 1918, `100.64/10`;
- link-local (169.254.169.254 included), multicast, ULA;
- the NAT64, 6to4 and Teredo forms, and IPv4-mapped forms of those;
- its own addresses.

It answers 403. The one exception is two clients relaying to each other: an own
address is a peer only at the relayed port of a live allocation. That is checked
when the permission or channel is made and again on every datagram, which is handed
over in memory, so SSH, 443 and the TURN listeners stay unreachable through the
relay.

`Config.AllowPeers` re-enables ranges (the tests use `127.0.0.0/8`).

## Bounds {#bounds}

All in `server.Config`: allocations per user and in total, lifetime cap,
per-allocation byte rate, relay port range, permissions and channels per allocation,
concurrent connections, idle and handshake timeouts, datagram size.

Permissions last 300 s and channels 600 s. Long-term credentials use NONCE expiry
(438).

## Secrets {#secrets}

Every secret is a file, refused when world-readable; none is a flag value or an
environment variable. Secrets rotate by repeating the flag: the first signs, the
others only verify.

## Browser origins and tokens {#origins-and-tokens}

- The WebSocket upgrade enforces the exact origin allowlist, scheme and port
  included.
- `/login` with an origin that is not allowed is a 400, never a redirect.
- `/callback` posts the session only to the allowlisted origin.
- A token is `authorization: Bearer …` metadata, never a URL.

## Rooms {#rooms}

A leaked room link lets a stranger knock, not enter; see
[Rooms and admission]({{< relref "rooms-and-admission.md" >}}).

## Untrusted input {#untrusted-input}

The `stun` decoder is strict and bounded: it refuses a length that overruns the
datagram, framing that is not 4-aligned, and more than `MaxAttributes` (64)
attributes. `FuzzDecode` and `FuzzChannelData` have a committed seed corpus under
`stun/testdata/fuzz`; CI also fuzzes the PROXY header parser, the credential parser
and keyring, and the session verifier.

## PROXY protocol {#proxy-protocol}

A PROXY header is accepted only from the prefixes given to `--proxy-protocol-from`;
a header from any other source is never honoured, and a connection from a listed
prefix without a valid header is closed.

## Reporting a vulnerability {#reporting}

Report privately through GitHub's private vulnerability reporting, from the
[Security tab](https://github.com/go-turn/turn/security) of go-turn/turn; do not
open a public issue for it. In scope: `turnd`, the relay policy, the credentials
and the rendezvous. Only the latest minor version is supported. The full policy is
`SECURITY.md` in that repository.
