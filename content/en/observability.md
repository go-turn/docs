---
title: "Observability"
weight: 65
---

# Observability {#observability}

`turnd` exports **traces, metrics and logs** over OTLP, with the official
OpenTelemetry Go SDK, to any collector (OpenTelemetry Collector, Grafana Alloy,
a vendor's endpoint). It is off, and costs nothing, until you give it an
endpoint. There is no Prometheus endpoint: the collector is the one place that
turns OTLP into whatever you scrape.

## Turning it on {#flags}

```sh
turnd … \
  --otel-endpoint https://otel.example.org:4318 \
  --otel-token-file /etc/turnd/otel-token
```

| Flag | Meaning |
|---|---|
| `--otel-endpoint` | `https://host:4318` for HTTP (the paths `/v1/traces`, `/v1/metrics`, `/v1/logs` are added), `host:4317` for gRPC. No user, password, query or fragment: a credential in a URL is refused, and never echoed in the error. |
| `--otel-protocol` | `http` (default) or `grpc` |
| `--otel-token-file` | a file holding the collector's bearer token, sent as `Authorization: Bearer …`. The file must not be readable by everybody. |
| `--otel-insecure` | allow `http://` or plaintext gRPC. A token is then refused unless the collector is on loopback. |
| `--otel-sample-ratio` | fraction of traces kept, 0 to 1 (default 1) |
| `--otel-service-name` | `service.name` (default `turnd`) |
| `--otel-trust-client-context` | continue a caller's W3C `traceparent` |

The resource carries `service.name`, `service.version` (the build version) and
`service.instance.id` (random, new at each start).

**The token is read from a file only.** `OTEL_EXPORTER_OTLP_HEADERS` and its
per-signal variants are ignored, and `turnd` says so at startup, so a collector
token never travels through the environment.

## Trace context {#trace-context}

`/login` and the WebSocket are public: anyone can send a `traceparent`. By
default `turnd` **restarts** the trace at the entry point, as W3C Trace Context
section 3.4 describes for a front gate, so a caller can neither join your traces
nor force sampling (section 7.2). Set `--otel-trust-client-context` only behind a
proxy or a client you control.

## Traces {#traces}

| Span | Attributes |
|---|---|
| `GET /login`, `GET /callback`, `GET /healthz` | `http.request.method`, `http.route`, `url.path`, `http.response.status_code`; Error status for a 5xx only; `turnd.outcome` on login and callback |
| every gRPC method, e.g. `turn.v1.Turnd/CreateRoom` | the otelgrpc attributes (`rpc.method`, `rpc.status_code`), plus `turnd.room.action` and `turnd.outcome` (`ok` or the gRPC code in lower case, `resource_exhausted`…) |
| `TURN allocate`, `TURN refresh`, `TURN create_permission`, `TURN channel_bind` | `turnd.method`, `turnd.transport` (`udp`, `tcp`, `tls`), `turnd.result`, `turnd.stun.error_code`, `turnd.policy` |

There is no span per relayed packet. The HTTP spans follow the HTTP semantic
conventions except that `url.query` is never recorded: the callback's query holds
the authorization code.

## Metrics {#metrics}

Units are UCUM. Every attribute has a small fixed set of values.

| Instrument | Kind, unit | Attributes |
|---|---|---|
| `turnd.relay.allocations` | gauge, `{allocation}` | |
| `turnd.relay.permissions` | gauge, `{permission}` | |
| `turnd.relay.channels` | gauge, `{channel}` | |
| `turnd.relay.requests` | counter, `{request}` | `turnd.method`, `turnd.transport`, `turnd.result` |
| `turnd.relay.policy.refusals` | counter, `{refusal}` | `turnd.policy`: `loopback`, `private`, `link_local`, `metadata`, `cgnat`, `multicast`, `translator`, `own_address`, `reserved`, `port_zero`, `family_mismatch` |
| `turnd.relay.auth.failures` | counter, `{failure}` | `turnd.surface` (`relay`, `api`), `turnd.reason` |
| `turnd.relay.bytes` | counter, `By` | `turnd.direction` (`client_to_peer`, `peer_to_client`) |
| `turnd.relay.packets` | counter, `{packet}` | `turnd.direction` |
| `turnd.relay.packets.dropped` | counter, `{packet}` | |
| `turnd.rooms` | gauge, `{room}` | |
| `turnd.guests` | gauge, `{guest}` | `turnd.state` (`pending`, `admitted`) |
| `turnd.rendezvous.operations` | counter, `{operation}` | `turnd.action`, `turnd.outcome` |
| `turnd.login.outcomes` | counter, `{login}` | `turnd.outcome` |
| `http.server.request.duration` | histogram, `s` | `http.request.method`, `http.route`, `http.response.status_code` |
| `rpc.server.*` | otelgrpc | |

One instrument per fact, so the questions you ask are queries:

- allocations created, refused, by transport and reason:
  `turnd.relay.requests` with `turnd.method="allocate"`, grouped by
  `turnd.transport` and `turnd.result`;
- the relay's `401` includes the first, unauthenticated request of every
  RFC 8489 exchange (the challenge): a rate, not an alarm, until it exceeds the
  allocations created;
- knocks, admits, denies: `turnd.rendezvous.operations` with `turnd.action` of
  `knock`, `admit`, `deny`;
- TURN credentials minted for a host or a room: the actions `credentials_host`
  and `credentials_room` with `turnd.outcome="ok"`;
- sessions issued and refused, by reason: `turnd.login.outcomes`.

The relay's packet and byte counters are read at each push, not on each packet:
the relay's hot path does the same work with telemetry on or off.

## Logs {#logs}

`turnd` logs with `log/slog`: one JSON object per line on stderr, which journald
keeps as is, with `trace_id` and `span_id` when a span is active. With telemetry
on, the same records are exported over OTLP through the OpenTelemetry slog
bridge, from Info up, after redaction. Every login outcome line is kept:
`login: session issued…`, `login refused: …`.

## What never leaves {#privacy}

No token, authorization code, cookie, credential, password, room ID or TURN
username, and no client or peer IP address, reaches a span, a metric or a log
record. In practice:

- attributes are fixed words, never request data;
- the address and port attributes the gRPC instrumentation adds on its own, and
  any attribute whose value is an IP address, are dropped before export;
- the debug records, which name peers, are never exported;
- the pseudonymous subject in the login line stays on stderr: the exported line
  reads `session issued to <redacted>`;
- the provider's error text is quoted and cut at 64 characters, as in the log.

The test suite runs the daemon against an in-process collector and searches every
request body for each of these strings.
