---
title: "Packages"
weight: 80
---

# Packages {#packages}

The module is `github.com/go-turn/turn`. Besides the `turnd` and `turnprobe`
commands it has these packages.

## `stun` {#stun}

The RFC 8489 message codec with the RFC 8656 TURN attributes and ChannelData
framing. Decoding is strict and bounded: it refuses a length that overruns the
datagram, framing that is not 4-aligned, and more than `MaxAttributes` (64)
attributes; attributes after `MESSAGE-INTEGRITY` are not visible.

It covers `MESSAGE-INTEGRITY` (HMAC-SHA1), `MESSAGE-INTEGRITY-SHA256`,
`FINGERPRINT`, the long-term key derivations (MD5 and SHA-256) and `USERHASH`.
Tested against the RFC 5769 vectors (decode, verify, byte-identical re-encode);
`FuzzDecode` and `FuzzChannelData` have a committed seed corpus under
`stun/testdata/fuzz`. SASLprep (OpaqueString) is the caller's job.

## `server` {#server}

The RFC 8656 TURN server: UDP relayed transport, reached over UDP, TCP or TLS (the
caller passes the sockets), plus the STUN Binding answer. Long-term credentials
(MD5, and SHA-256 when the client asks) with NONCE expiry (438); permissions last
300 s, channels 600 s.

```go
srv, err := server.New(server.Config{
	Realm:   "turn.example.org",
	Auth:    func(username string) (password string, ok bool) { /* wrap creds.Verify */ },
	RelayIP: netip.MustParseAddr("203.0.113.5"),
})
go srv.ServePacketConn(udpConn)         // UDP clients
go srv.ServeListener(tcpListener)       // TCP clients
go srv.ServeTLS(tcpListener443, tlsCfg) // turns:
```

What the server refuses to relay to, and its bounds, are in the
[Security policy]({{< relref "security.md" >}}).

`interop/` is the judge: `github.com/pion/turn/v4` (test-only) allocates, permits,
binds a channel and round-trips data through the server over UDP and TCP.
`go test -bench . ./server` prints allocations/s and relayed packets/s; the
comparison with coturn on one core is still to do.

## `creds` and `session` {#creds-and-session}

`creds` is the credential scheme and its keyring (the
[TURN REST API](https://datatracker.ietf.org/doc/html/draft-uberti-behave-turn-rest-00)
usernames `<expiry>:<tag>.<subject>`); `session` verifies the session token a login
produces.

## `rendezvous/client` {#rendezvous-client}

The Go client for the rooms; see [The gRPC API]({{< relref "grpc-api.md#the-go-client" >}}).
