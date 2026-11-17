---
title: "Deploying"
weight: 20
---

# Deploying {#deploying}

One host, one binary:

```sh
CGO_ENABLED=0 go install github.com/go-turn/turn/cmd/turnd@latest
```

## Ports {#ports}

Open these at the host firewall **and** at the cloud security group:

| port | for |
|---|---|
| UDP 3478 | STUN and TURN over UDP (`--stun-listen :3478`) |
| TCP 3478 | TURN over TCP (`--turn-tcp-listen :3478`) |
| TCP 5349 | TURN over TLS (`--turns-listen :5349`), for networks that block UDP |
| UDP range | relayed addresses (`--relay-port-min 49160 --relay-port-max 49999`) |
| TCP 443 | HTTPS: `/login`, `/callback`, `/grpc`, through the host's reverse proxy to `--listen 127.0.0.1:8080` (the proxy must pass WebSocket upgrades) |

To share 443 between that HTTPS site and TURN over TLS, see
[TURN over TLS on 443]({{< relref "tls-on-443.md" >}}).

## The HTTP half {#the-http-half}

```sh
turnd --issuer https://login.example.org --client-id turnd \
  --client-secret-file /etc/turnd/client-secret \
  --session-key-file /etc/turnd/session.key \
  --turn-secret-file /etc/turnd/turn.secret \
  --allowed-origin https://go-tex.github.io \
  --public-url https://turn.example.org --listen 127.0.0.1:8080 \
  --turn-uri 'turns:turn.example.org:443?transport=tcp'
```

Register `<public-url>/callback` as the client's redirect URI at the issuer
(the one named by `--client-id`).

## The relay {#the-relay}

`turnd` starts the relay when a listen flag is set:

| flag | |
|---|---|
| `--relay-ip` | required; `--relay-advertise-ip` behind a NAT |
| `--stun-listen :3478` | STUN and TURN over UDP |
| `--turn-tcp-listen :3478` | TURN over TCP |
| `--turns-listen :443` | TURN over TLS, with `--tls-cert-file` and `--tls-key-file` |
| `--relay-port-min` / `--relay-port-max` | the range of relayed UDP ports |
| `--proxy-protocol-from <prefix>` | repeatable; accept a [PROXY protocol v2](https://github.com/haproxy/haproxy/blob/master/doc/proxy-protocol.txt) header on TCP and TLS connections from that prefix |

The relay's credential hook is the keyring's `Auth`, so the minted
`<expiry>:<tag>.<subject>` usernames are what it accepts (the tag says which of
the rotating secrets signed it).

## Secrets {#secrets}

Every secret is a file, refused when world-readable; none is a flag value or an
environment variable. Make them readable by the service account only:

| flag | content |
|---|---|
| `--session-key-file` | 32+ random bytes |
| `--turn-secret-file` | 16+ random bytes |
| `--client-secret-file` | the issuer's client secret |

Repeat `--session-key-file` / `--turn-secret-file` to rotate: the first signs, the
others only verify.

## After deploying {#after-deploying}

Prove the relay end to end without an OpenID Connect login:
[Checking a deployment]({{< relref "checking-a-deployment.md" >}}).
