---
title: "Checking a deployment"
weight: 40
---

# Checking a deployment {#checking-a-deployment}

A deployed relay can be proven end to end without an OpenID Connect login.

## Mint credentials on the host {#mint-credentials}

On the host, mint short-lived credentials with the keyring turnd uses (the first
`--turn-secret-file` signs; the TTL is capped at 1h). The username is printed; the
password goes to a new `0600` file, never to stdout or the command line:

```sh
turnd mint-credentials --turn-secret-file /etc/turnd/turn.secret \
  --subject probe --ttl 10m --password-out ./probe.pw   # prints <expiry>:<tag>.probe
```

## Run turnprobe {#turnprobe}

From anywhere that can reach the relay, run `turnprobe`. For each `--server` it
sends a STUN Binding (printing the mapped address), opens two allocations, permits
each other's relayed address, and sends a datagram from relay A to relay B and
back; that proves relaying, with nothing else needed. A `turns:` certificate is
verified against the system roots with SNI set to the host.

```sh
go install github.com/go-turn/turn/cmd/turnprobe@latest
turnprobe --username '1763400000:ab12.probe' --password-file ./probe.pw \
  --realm turn.example.org \
  --server 'turn:turn.example.org:3478?transport=udp' \
  --server 'turn:turn.example.org:3478?transport=tcp' \
  --server 'turns:turn.example.org:443?transport=tcp'
```

## Reading the result {#reading-the-result}

```
PASS turn:turn.example.org:3478?transport=udp mapped=198.51.100.7:51234 relay-a=203.0.113.5:49160 relay-b=203.0.113.5:49161 roundtrip=3ms
FAIL turns:turn.example.org:443?transport=tcp stage=connect: tls: failed to verify certificate: ...
```

One line per server, `PASS` or `FAIL` with the stage that failed: `connect`,
`binding`, `allocate`, `permission`, `relay A->B`, `relay B->A`. The exit status is
1 if any failed.

`--realm` is turnd's `--realm`; a mismatch is reported at `allocate`.
