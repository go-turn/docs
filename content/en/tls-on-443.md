---
title: "TURN over TLS on 443"
weight: 30
---

# TURN over TLS on 443, beside HTTPS {#turn-over-tls-on-443}

TURN over TLS can share 443 with the HTTPS site when Caddy demultiplexes by SNI
with [caddy-l4](https://github.com/mholt/caddy-l4). Caddy terminates TLS for
`turn.example.org` itself and proxies plain TCP to turnd with a **PROXY protocol
v2** header, so turnd still sees the real client address (XOR-MAPPED-ADDRESS,
quotas and logs).

## Build Caddy {#build-caddy}

```sh
xcaddy build --with github.com/mholt/caddy-l4@v0.1.3
```

The version is pinned because caddy-l4 calls itself experimental.

## The Caddyfile {#the-caddyfile}

```
{
    servers :443 {
        listener_wrappers {
            layer4 {
                @turn tls sni turn.example.org
                route @turn {
                    tls
                    proxy {
                        proxy_protocol v2
                        upstream tcp/127.0.0.1:3478
                    }
                }
            }
            tls
        }
    }
}

# A site block makes Caddy obtain the certificate that the `tls` handler above
# presents for turn.example.org.
turn.example.org {
    respond "turnd relay" 200
}
```

## turnd's side {#turnds-side}

```sh
turnd --turn-tcp-listen 127.0.0.1:3478 --proxy-protocol-from 127.0.0.1/32 …
```

Clients use `turns:turn.example.org:443?transport=tcp`.

## What the PROXY header allows {#what-the-proxy-header-allows}

The header is accepted only from the listed prefixes: a connection from one of
them without a valid header is closed, and a header from any other source is
never honoured.

## What has been tested {#what-has-been-tested}

The Caddyfile follows caddy-l4's `docs/examples/combining_apps.md`; it has not
been run against a Caddy by the project, only the header handling is tested (with
pion's TCP client behind a proxy that prepends a v2 header).

The separate `--turns-listen` (TLS handled by turnd, port 5349 or 443 on a second
address) stays available for a host without Caddy; there the PROXY header, if any,
precedes the ClientHello.
