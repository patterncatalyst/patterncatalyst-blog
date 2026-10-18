---
title: "HTTP/3 Fixes TCP, Not Your Protocol: Why gRPC Doesn't Get a Free Upgrade"
date: 2026-10-18
author: "Pattern Catalyst"
tags: [grpc, http3, quic, http2, transport, api-design]
categories: [api-design]
canonical_project:
  name: "API Patterns & Practices: gRPC"
  repo: "patterncatalyst/api-patterns-and-practices-grpc"
  url: "https://patterncatalyst.github.io/api-patterns-and-practices-grpc/"
excerpt: "The moment someone hears HTTP/3 fixes TCP's head-of-line blocking, the next sentence is usually that gRPC gets faster. The first half is true and the second doesn't follow. QUIC fixes TCP's problems, and gRPC is defined over HTTP/2 specifically, so gRPC-over-HTTP/3 is a separate binding neither reference stack ships yet."
---

Sit in enough architecture reviews and you will hear this pair of sentences
delivered as one thought: "HTTP/3 fixes TCP's head-of-line blocking, so once we
move to it, gRPC gets faster." The first half is correct. The second does not
follow, and teams that conflate them plan around a speedup that will not arrive.
QUIC fixes problems that belong to TCP. gRPC's performance characteristics belong
to HTTP/2, a different layer, and "gRPC over HTTP/3" is a distinct piece of
engineering that has to be specified and built on its own. As of early 2026,
neither of the two reference implementations behind most production gRPC ships it.

This post is mined from a gRPC course whose every hands-on chapter ships the same
Protocol Buffers contract in both Quarkus and Python. Its Chapter 24 states
outright that gRPC-over-HTTP/3 does not exist on these stacks yet, rather than
hedging around a hypothetical, then builds a demo that keeps the two transports
strictly separate.

## What QUIC fixes

HTTP/1.1 and HTTP/2 both ride on TCP, and TCP delivers bytes to the application
in strict order. That ordering guarantee is the whole problem. HTTP/2 multiplexes
many independent streams over one TCP connection, but TCP underneath has no idea
those streams are independent. It sees one ordered byte sequence. Lose a single
segment, and every stream sharing that connection stalls behind the
retransmission, including the streams whose own bytes already arrived intact.
This is head-of-line blocking, and it is a *transport-level* block: it has nothing
to do with HTTP and everything to do with TCP's ordering promise.

QUIC, the transport HTTP/3 runs on, moves to UDP and tracks each stream's
delivery independently. A lost packet belonging to one stream stalls that stream's
retransmission and nothing else. The others keep flowing.

{% include excalidraw.html
   file="http3-fixes-tcp-not-grpc"
   alt="Two side-by-side panels. The left panel, HTTP/2 over TCP, shows three streams A, B, and C feeding one TCP box doing strict in-order delivery; stream B has a lost segment, and an emphasis box reads that one lost segment stalls all three streams, a transport-level head-of-line block. The right panel, HTTP/3 over QUIC, shows the same three streams feeding a QUIC box doing per-stream delivery over UDP; stream B retransmits alone while A and C flow normally, and an emphasis box reads that a lost packet stalls only its own stream, with no cross-stream head-of-line blocking."
   caption="One lost segment stalls every multiplexed HTTP/2 stream sharing a TCP connection. QUIC's independent per-stream delivery means a lost packet only stalls the one stream it belongs to." %}

QUIC adds two more properties that show up in almost every HTTP/3 pitch. **0-RTT
resumption** lets a client that has talked to a server before skip a full round
trip when it reconnects. **Connection migration** lets a client's address change,
a phone moving from Wi-Fi to cellular, without tearing down the connection,
because QUIC identifies a connection by a connection ID rather than by the client
IP and port tuple that TCP is stuck with.

All three are real improvements, and every one fixes something TCP got wrong. None
of them says a word about which application protocol rides on top.

## Why "gRPC gets faster" doesn't follow

HTTP/3 is HTTP semantics, the same request and response model, the same status
codes, the same header idea, carried over QUIC instead of over TCP. The
application protocol above it still has to be defined in terms of what the
transport offers. And gRPC's wire format is defined in terms of HTTP/2
specifically. It uses HTTP/2 trailers to carry the final status code, HTTP/2
flow control for per-stream backpressure, and one HTTP/2 stream per RPC. Those
are not incidental implementation details a library can swap under the hood; they
are the gRPC-over-HTTP/2 specification.

So "gRPC over HTTP/3" is not a flag you flip on an existing HTTP/2 stack. It is a
new transport binding that has to be specified and implemented separately, the
same way gRPC-over-HTTP/2 had to be specified and implemented once. That binding
does not exist natively in grpc-java or grpcio, the two implementations behind
every Quarkus and Python example in the source course. Neither project ships a
QUIC-based transport a `GrpcService` or a `grpc.aio.server()` can bind to. This is
not a quirk of two libraries; it reflects where the broader gRPC ecosystem sits
today.

{% include excalidraw.html
   file="grpc-over-http3-where-it-stands"
   alt="Three stacked tiers. Tier 1, native gRPC over HTTP/2, shows grpc-java on Quarkus and grpcio on Python both marked available, labeled the book's default and CI-gated every chapter, with an emphasis box noting every gRPC call in the book travels here. Tier 2, HTTP/3 as a transport not gRPC, shows Caddy with automatic HTTP/3 over TLS and an aioquic probe doing a QUIC handshake and h3 GET, both available, with an emphasis box noting it proves the transport runs but never makes a gRPC call. Tier 3, gRPC directly over HTTP/3, shows grpc-java and grpcio both marked as having no native h3 transport, with an emphasis box noting it is not available in either stack as of January 2026."
   caption="Three tiers, only two of them real today: native gRPC over HTTP/2 is the hard-gated default; HTTP/3 is demonstrable as a transport on its own; gRPC directly over HTTP/3 is a binding neither reference implementation ships yet." %}

The consequence for a demo is that you cannot bolt HTTP/3 onto a gRPC
server and point at the result. The course's Chapter 24 example respects that by
keeping the two transports in entirely separate services. Two lean servers answer
one RPC over native gRPC on HTTP/2, the only transport either stack has. A third,
unrelated service exists only to give a QUIC client something real to talk to.

## Proving the transport without overstating it

That third service is Caddy, serving one static JSON file over TLS. Caddy turns
HTTP/3 on automatically for any site it serves over TLS, with no separate
directive, which makes it a convenient real QUIC endpoint. The entire
configuration is a plain file server:

```caddyfile
# examples/24-cross-protocol-http3/caddy/Caddyfile
# A transport demo, not a gRPC server. Nothing here speaks gRPC; Caddy is
# not proxying to either RoutingService. It exists only so the probe has a
# real HTTP/3-over-QUIC endpoint to complete a handshake against.
{
	auto_https disable_redirects
}

caddy:8443, localhost:8443 {
	tls internal
	root * /srv/www
	file_server
	header Content-Type application/json
}
```

The client is a small `aioquic` probe, used entirely independently of `grpcio`.
It advertises the `h3` ALPN token, which is how a client tells a server during
the TLS handshake which application protocol it wants (`h2` for HTTP/2, `h3` for
HTTP/3), drives one GET, and records two facts: did the QUIC handshake complete,
and did the response come back `200`.

```python
# examples/24-cross-protocol-http3/python/app/h3_probe.py
# Proves HTTP/3 the transport. There is no gRPC call anywhere in this module.
async def probe(host: str, port: int, path: str, timeout: float) -> int:
    configuration = QuicConfiguration(alpn_protocols=H3_ALPN, is_client=True)
    # Demo-only: Caddy's `tls internal` mints a fresh CA on every start, so
    # verification is off here. This script proves a protocol can run, not
    # that a certificate chain is trustworthy.
    configuration.verify_mode = ssl.CERT_NONE

    async with connect(
        host, port, configuration=configuration, create_protocol=H3ProbeProtocol
    ) as protocol:
        probe_protocol = cast(H3ProbeProtocol, protocol)
        await probe_protocol.get(f"{host}:{port}", path, timeout)
    # ...
```

When it runs, it reports what it did and names what it did not do:

```
h3_probe: QUIC handshake_completed=True alpn=h3 status=200
h3_probe: HTTP/3 transport confirmed over QUIC — this is the transport,
          not gRPC-over-HTTP/3 (no gRPC call was made)
```

QUIC rides on UDP, and UDP is blocked between containers on far more CI runners
and corporate networks than TCP is. The example treats this probe as
capability-gated: it retries a few times to absorb Caddy's cold start, and if it
still can't complete, it logs a clear `SKIP` with a documented fallback and lets
the build continue. Whether a sandbox routes UDP between two containers is a
property of the environment, not of the code, so wiring a build failure to it
would signal a code defect that is not there.

Meanwhile the gRPC call the course measures runs over HTTP/2 on both stacks, and
it is fast without any of this:

```
Quarkus:  rps=1888.10  p99=12.58ms  errors=0
Python:   rps=5117.61  p99=4.64ms   errors=0
```

That server is the leanest in the whole course, one RPC and nothing else, because
the chapter's subject is the transport a call rides on, not a new capability:

```python
# examples/24-cross-protocol-http3/python/app/routing_service.py
# Only GetQuote is overridden; the generated base class returns UNIMPLEMENTED
# for every other method. HTTP/2 is the only transport this call ever uses.
class RoutingServiceServicer(routing_pb2_grpc.RoutingServiceServicer):
    async def GetQuote(self, request, context):
        quote = compute_quote(request.cargo)
        leg = types_pb2.Leg(
            carrier=_CARRIER, origin=request.origin, destination=request.destination)
        route = types_pb2.Route(
            id=f"rte_{request.origin.un_locode}-{request.destination.un_locode}",
            legs=[leg], price=quote)
        return routing_pb2.GetQuoteResponse(quote=quote, route=route)
```

## The choice that happens before any of this

Step back one level, because "should we move to HTTP/3" is rarely the first
transport question a team faces. The first one is how a new streaming
interaction should travel across the wire at all, and gRPC is one of at least
three mature answers. Each optimizes for a different caller.

- **gRPC over HTTP/2** gives up nothing on the server-to-server path: a Protobuf
  contract generated into real client and server code, all four RPC shapes
  including bidirectional streaming, and end-to-end flow control where a slow
  consumer throttles the producer on its own stream. The cost is reach. A browser
  cannot speak native gRPC, and anyone debugging without the `.proto` in hand is
  working blind.
- **REST over HTTP/2**, meaning server-sent events or chunked transfer over a
  conventional REST API, trades the typed contract for universal reach. Any
  browser's `fetch` or `EventSource` consumes it with zero extra tooling and any
  engineer with `curl` can watch it live. What it does not give you is a second
  direction; SSE and chunked responses are server-to-client only, and
  backpressure is yours to build.
- **WebSockets** go the other way: full-duplex, free-form messages, native in
  every browser, and no schema anywhere in the protocol. That absence is the
  design point, a raw bidirectional pipe, but it means message shapes live in
  documentation and convention, with no `buf breaking` equivalent to catch an
  incompatible change before it reaches a client.

{% include excalidraw.html
   file="three-transports-who-calls"
   alt="A three-column comparison. The gRPC over HTTP/2 column lists a Protobuf CI-enforced contract, all four RPC shapes, browser reach needing grpc-web and a proxy, and HTTP/2 flow control; best when both ends are services you own. The REST over HTTP/2 column lists an OpenAPI hand-maintained contract, one-way SSE or chunked streaming, native browser fetch and EventSource reach, and application-level backpressure; best when a browser needs a one-way feed. The WebSockets column lists no built-in contract, full-duplex free-form messages, native WebSocket API reach, and application-level backpressure; best when the UI needs fluid full-duplex. A note across the top reads that the deciding question is who calls and which way data flows, not a throughput number."
   caption="The same questions asked of all three transports: what is the contract, which way does data flow, who can call without extra tooling, and who owns backpressure. The deciding factor is who is on the other end, not a benchmark." %}

None of the three is categorically faster. A payments service calling another
internal service wants gRPC's typed contract and controls both ends of the wire.
A public dashboard wants REST-over-HTTP/2's zero-tooling reach for a one-way
feed. A collaborative editor wants WebSockets' full duplex and accepts the
schema-discipline cost because its message shapes change too often to justify
generated stubs. The deciding question is who calls and which way data flows, and
a throughput number almost never settles it.

## What to carry into the next planning meeting

Do not let an HTTP/3 timeline move a transport decision. HTTP/2 is the gRPC
transport everywhere today, on every mainstream stack, and that stays true until
grpc-java and grpcio (or whichever implementation you depend on) ship a native
QUIC binding. When that lands it is worth revisiting, not before. QUIC's fix for
head-of-line blocking accrues to HTTP/3 and to protocols built directly on QUIC;
it does not reach a protocol defined over HTTP/2. Keep the layer where the fix
lives separate from the layer where your protocol lives, and the two questions
stop collapsing into one wrong answer.

---

**Source project.** This post is mined from
[Chapter 24, cross-protocol choices and HTTP/3](https://patterncatalyst.github.io/api-patterns-and-practices-grpc/docs/24-cross-protocol-http3/)
of [API Patterns & Practices: gRPC](https://patterncatalyst.github.io/api-patterns-and-practices-grpc/),
a contract-first gRPC course that ships one Protocol Buffers shipping-and-cargo
contract in both Quarkus and Python, validated under Buf. The files shown here:
[`Caddyfile`](https://github.com/patterncatalyst/api-patterns-and-practices-grpc/blob/r1.0/examples/24-cross-protocol-http3/caddy/Caddyfile),
[`h3_probe.py`](https://github.com/patterncatalyst/api-patterns-and-practices-grpc/blob/r1.0/examples/24-cross-protocol-http3/python/app/h3_probe.py),
and
[`routing_service.py`](https://github.com/patterncatalyst/api-patterns-and-practices-grpc/blob/r1.0/examples/24-cross-protocol-http3/python/app/routing_service.py);
the chapter's [`verify.sh`](https://github.com/patterncatalyst/api-patterns-and-practices-grpc/blob/r1.0/examples/24-cross-protocol-http3/verify.sh)
runs all three services, measures the gRPC call with `ghz`, and runs the HTTP/3
probe without failing the build on it. Full source is on
[GitHub](https://github.com/patterncatalyst/api-patterns-and-practices-grpc).
More from the catalog on the [Pattern Catalyst hub]({{ site.hub_url }}).
