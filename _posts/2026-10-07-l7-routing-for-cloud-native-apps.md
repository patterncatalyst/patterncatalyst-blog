---
title: "L7 Routing Is a Four-Layer Decision Chain, Not an Appliance in a Rack"
date: 2026-10-07
author: "Pattern Catalyst"
tags: [cloud-native, l7-routing, kubernetes, api-gateway, service-mesh]
categories: [cloud-native]
canonical_project:
  name: "Cloud-Native Design Patterns"
  repo: "patterncatalyst/cloud-native-design-patterns"
  url: "https://patterncatalyst.github.io/cloud-native-design-patterns/"
excerpt: "L7 routing in 2026 isn't a box in a rack — it's layered software (edge, gateway, mesh, app) making a decision on every HTTP and gRPC envelope, and knowing which layer owns which decision is the whole game."
---

Layer-7 routing used to mean a load balancer appliance in a rack. It doesn't anymore. It's
layered software running inside Kubernetes that makes a decision on every HTTP and gRPC
envelope crossing the network — both north/south (clients into the system) and east/west
(service to service). Most of those decisions, the ones that actually move the needle on
reliability and velocity, happen east/west, not at the edge.

Each layer — edge, gateway, mesh, and the application itself — can see and decide different
things, and most of the interesting work happens deeper than teams expect. What follows is
the decision chain layer by layer: what the mesh can do with TLS, sticky sessions, and
traffic steering; what only the service itself can decide with rule-driven routing; and why
east/west is where most of it lives.

## L4 versus L7 — what each layer can decide on

A Layer-4 load balancer sees *connections*: IPs, ports, the TCP 5-tuple. It picks a backend
and hash-balances, but it can't read URLs, headers, cookies, methods, or gRPC metadata. A
Layer-7 router sees the whole *conversation*: host, path, headers, cookies, JWT claims, query
parameters, the HTTP method, the gRPC `service.method` pair. That full envelope is the menu
every routing decision in this post draws on.

The one nuance worth holding onto is TLS: an L4 balancer can route by SNI *without*
terminating TLS — cheap, opaque — while an L7 router typically terminates TLS so it can read
everything else — richer, slightly slower.

{% include excalidraw.html
   file="l7-four-layer-stack"
   alt="North/south: a client passes through edge/ingress (TLS, host/path), then the API gateway (auth, rate limit), then a mesh sidecar (mTLS, retries), then the service (in-app rules). East/west: service A and service B talk through mesh sidecars that add mTLS, outlier detection, and locality routing. Each layer makes a different kind of decision and emits a trace span."
   caption="L7 routing is a four-layer stack; the mesh applies the same routing to every internal call, not just at the edge" %}

## The four-layer stack

L7 routing lives in four layers, and each one is the right home for a *different kind* of
decision. A request is really a chain of independent L7 decisions across all four.

The **edge / ingress** layer (HAProxy, Nginx, Envoy, Traefik) is where TLS terminates and
coarse host/path routing happens. The **API gateway** (Kong, Apigee, an Istio gateway, Spring
Cloud Gateway) handles product-level concerns — authentication, rate limiting,
transformation. The **service mesh** (Istio with Envoy sidecars, or Linkerd) governs
east/west traffic, giving every internal call mTLS, retries, outlier detection, and
locality-aware load balancing. And when a decision depends on business rules the platform
can't know, routing pushes into the **app** itself. Each layer can refuse, transform, or
branch the request, and each emits a trace span — the request's journey is observable end to
end.

## Content-based routing

HAProxy and Nginx are the workhorses of the edge — mature, fast, predictable. HAProxy reads
the envelope with ACLs (`path_beg` for URL prefixes, `hdr` for headers, `content-type` for
the body kind) and `use_backend` dispatches on them; Nginx uses `location` blocks and the
`map` directive to turn a header value into an upstream pool. Envoy is the modern sidecar
choice because it speaks HTTP/2 and gRPC natively.

But the form most teams reach for today is the mesh's: the same content-based primitives
expressed as Kubernetes CRDs that Istio compiles down to Envoy config across every sidecar.

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata: { name: orders }
spec:
  hosts: [orders]
  http:
  - match:                                    # gRPC by service.method
    - uri: { prefix: /acme.orders.v1.OrderService/ }
    route: [{ destination: { host: orders, subset: v2 } }]
  - match:                                    # by header
    - headers: { x-internal: { exact: "1" } }
    route: [{ destination: { host: orders, subset: v2 } }]
  - match:                                    # by cookie
    - headers: { cookie: { regex: ".*canary=on.*" } }
    route: [{ destination: { host: orders, subset: v2 } }]
  - route: [{ destination: { host: orders, subset: v1 } }]   # default
```

Three match rules — by URI prefix (which is how you route on a gRPC `service.method`, since
gRPC paths look like `/package.Service/Method`), by header, and by cookie — each sending to a
different subset, with a default route at the bottom that's *critical*: without it, unmatched
requests have nowhere to go. The CRD lives in Git, gets reviewed like any other change, and
Istio reconfigures every sidecar within seconds.

The runnable example backing this post uses a lighter touch at the edge — plain Envoy
weighted clusters and a header override, no CRDs required:
[`examples/22-l7-routing/envoy/envoy.yaml`](https://github.com/patterncatalyst/cloud-native-design-patterns/blob/main/examples/22-l7-routing/envoy/envoy.yaml).

## TLS termination

TLS at the edge comes in three modes. **Terminate-and-decrypt** is the common case — it lets
the router do everything L7. **Terminate-and-re-encrypt** is the most secure default in a
mesh, since mTLS inside is automatic. **Passthrough** is for traffic you must not see (client
mTLS for regulated flows): route by SNI only, decrypt nothing. cert-manager is the standard
answer for public certificates; SPIFFE-style identity and Envoy's SDS handle
service-to-service certs automatically inside the mesh. A handshake is milliseconds, so
amortise it with session resumption, TLS 1.3, and HTTP/2 multiplexing.

## Sticky sessions

There are three affinity modes, and the default is the one you want. **No affinity** sends
any request to any pod — even load, and client state lives elsewhere (Redis, the JWT, the
database). **Cookie-based** stickiness is the fallback when something stateful must stay on
one pod: the balancer sets a cookie and routes subsequent requests back to the same pod — but
that ties the session to one pod's lifetime, so a restart loses it. **IP-hash** needs no
cookie but is fragile behind NAT and mobile networks, where many users share an address and
load skews.

The real cost is that stickiness fights elastic scaling, fast restarts, and graceful
shutdown — the entire cloud-native delivery model. Treat it as a deliberate choice for a known
constraint, never a default, and prefer to externalise state (Redis for session data, JWT
claims for identity, a pub/sub backplane for fan-out) so the balancer can return to
no-affinity.

## Intelligent traffic steering

Six familiar release shapes are all the same L7 primitive with different inputs and weights:
canary (a small weighted slice to a new version), blue/green (two environments flipped
atomically), A/B (a stable cohort mapped to a variant by a user id), header-based dark
launches (route opted-in users by a header), geo/latency (EU users to EU pods), and
shadow/mirror (copy a percentage of real traffic to a new version without returning its
responses). The canonical one is the weighted canary: a `VirtualService` splits traffic by
weight, and a `DestinationRule` defines the subsets by pod label.

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata: { name: orders }
spec:
  hosts: [orders]
  http:
  - route:
    - destination: { host: orders, subset: v1 }
      weight: 90                              # 90% on the stable version
    - destination: { host: orders, subset: v2 }
      weight: 10                              # 10% on the canary
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata: { name: orders }
spec:
  host: orders
  subsets:
  - name: v1
    labels: { version: v1 }
  - name: v2
    labels: { version: v2 }
```

Ramping the canary is changing the weights in Git — typically 10, 25, 50, 100 percent with
metric gates between steps — and a controller like Flagger or Argo Rollouts flips the weights
back in seconds if error rate or latency on `v2` crosses a threshold. Header-based steering
adds surgical precision: an upstream identity proxy sets a header for opted-in users and the
`VirtualService` routes *those* users to `v2`, on real production data with zero blast
radius.

Payload modification is the cheap cousin — header inject/strip and URL rewrites are nearly
free because the router is already parsing the line and headers (this is how the W3C
`traceparent` header propagates, and how a gateway validates a JWT once and injects trusted
claims downstream). Body transformation is the expensive one: it forces the router to buffer
the full payload, killing streaming and risking memory under load. The rule of thumb: if you
can do it in the headers, do it in the headers.

## Routing inside the app by business rules

There's a threshold where routing leaves the network and enters the service. If the decision
fits in a few YAML lines and depends only on the envelope, do it at the mesh. If it depends
on the *payload*, requires a decision table that domain experts edit, or involves many
intersecting conditions — think routing a healthcare message by patient class and encounter
type — push it into a rule engine inside the app, behind the mesh, not in a network CRD. The
discipline is "data, not code": the routing logic lives in an externalised ruleset the domain
experts own, separate from the dispatcher, and changes without redeploying the service.

Each ecosystem composes this from its own rule engine — a Rete forward-chaining engine in
most cases — feeding a small dispatcher. The ruleset reads as a set of "when … then …"
statements that set a list of destinations:

```python
# orders_rules.py  —  edited by domain experts, not developers (durable-rules)
from durable.lang import ruleset, when_all, m

with ruleset("orders"):

    @when_all((m.amount_cents > 50_000) & (m.tier == "VIP"))
    def vip(c):                              # high-value VIP orders
        c.m.where_to = ["kafka:orders.priority", "http:notify-vip"]

    @when_all(m.region == "EU")
    def eu(c):                               # European orders → EU region
        c.m.where_to = ["kafka:orders.eu"]

    @when_all(+m.order_id)                   # catch-all (lowest priority)
    def default(c):
        if not getattr(c.m, "where_to", None):
            c.m.where_to = ["kafka:orders.default"]
```

The dispatcher enriches the input with whatever context the rules need, evaluates the
ruleset, and fans out to each destination over the right transport. The pattern is the
classic content-based router from the enterprise-integration literature — the value is that
the *decision* is data a domain expert edits, not code a developer redeploys.

The runnable version of this dispatcher — a FastAPI service that routes VIP orders to a
priority topic and lets you change the threshold at runtime with no redeploy — is here:
[`examples/22-l7-routing/python/router-service/main.py`](https://github.com/patterncatalyst/cloud-native-design-patterns/blob/main/examples/22-l7-routing/python/router-service/main.py).

## The cost of L7 hops

Every L7 capability costs time, and it's a budget you spend before your service even runs.
TLS termination is sub-2ms, path/header routing sub-1ms, JWT validation a few ms, a WAF a few
more, each mesh sidecar about a millisecond, a rate-limit Redis lookup a couple of ms. Body
inspection is the outlier at roughly 5–50ms depending on size.

The working heuristic is to stay under about 10ms of total L7 on hot paths — and with four
layers (edge, gateway, sidecar out, sidecar in) that means every layer has to earn its place.
The common pitfalls follow from this: too many hops nobody measured until p99 spiked;
expensive regex matches; sticky sessions fighting the platform; body inspection blowing the
budget; per-request auth lookups cascading under load; and timeout misalignment — the
gateway-to-mesh-to-service version of a retry storm. The fix is the same one that fixes retry
storms: bound retries, align timeouts top-down, and give the deepest layer the strictest
budget.

## East/west — L7 between services

Most L7 decisions in a microservices system aren't at the edge; they happen east/west, every
time one service calls another. The mesh's sidecar pair handles that traffic with no app
code: mTLS (encryption *and* identity), HTTP/2 multiplexing, native gRPC, retries with
backoff and jitter, outlier detection that ejects misbehaving pods, locality-aware load
balancing that prefers same-zone pods, and per-call traces. Same Istio CRDs as the edge,
pointed at an internal host:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata: { name: payments }
spec:
  host: payments.acme.svc.cluster.local
  trafficPolicy:
    connectionPool:
      http: { http2MaxRequests: 1000, maxRequestsPerConnection: 10 }
    outlierDetection:                              # eject bad pods
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
    loadBalancer:
      localityLbSetting: { enabled: true }         # prefer same zone
  subsets:
  - { name: v1, labels: { version: v1 } }
  - { name: v2, labels: { version: v2 } }
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata: { name: payments }
spec:
  hosts: [payments.acme.svc.cluster.local]            # internal DNS — east/west
  http:
  - route:
    - { destination: { host: payments, subset: v1 }, weight: 95 }
    - { destination: { host: payments, subset: v2 }, weight: 5 }
```

That last block is a canary release of a backend service no external client knows exists —
something you simply can't do through an edge load balancer.

gRPC east/west has a few sharp edges worth naming: route on the gRPC URI prefix for
per-method canaries; prefer least-request load balancing over round-robin, because gRPC
connections are long-lived and round-robining connections (rather than requests) can starve
backends; be careful retrying streaming RPCs, since replaying a half-finished stream redoes
half the work; and remember gRPC returns its real status in HTTP trailers, so outlier
detection has to read the trailer — a `200` with a non-OK gRPC status is a failure, and a
naive check would call a failing service healthy.

<!--
  Additional figures from the source appendix — the L7 journey, sticky-session
  affinity modes, the six traffic-steering shapes, and east/west sidecar
  mechanics — are available as paired SVG + .excalidraw assets in the source
  project if this post grows into a series. Pattern for adding one here:

  {% include excalidraw.html
     file="<name>"
     alt="<describe what the diagram shows>"
     caption="<figure caption>" %}
-->

## When to use which layer

Each layer has a job it's good at, and pushing work to the wrong one is where complexity
comes from. Edge/ingress for the things every client crosses (TLS, host/path). API gateway
for product-shaped policy (auth, rate limits, plans). Mesh for east/west and everything
internal. In-app for business rules. The antipattern this prevents is the slow growth of
baroque `VirtualService` configs that encode rules domain experts need to maintain — if it's
a business rule, it belongs in a rule engine inside a service, not a network CRD. And the
organisational failure to avoid is the cluster that ends up running three ingress
controllers because three teams each picked one: choose one ingress, one gateway, one mesh,
manage routing through GitOps, watch certificate rotation, and load-test the failure paths.

### Prove it yourself

Apply the canary `VirtualService`/`DestinationRule`, drive a few hundred requests with `hey`,
and confirm the split lands near the configured weight — roughly 10% of responses carrying
the `v2` build marker (a response header or version field), the rest `v1`. Bump the weight in
Git, re-apply, and watch the ratio move within seconds. Then flip a request's `x-internal`
header and confirm it goes to `v2` every time regardless of weight — header match beats
weight.

For the in-app router: post one VIP order over the configured threshold and one ordinary
order, and confirm the VIP order landed on `orders.priority` (and triggered the `notify-vip`
HTTP call) while the ordinary one fell through to `orders.default` — then change only the
ruleset, not the dispatcher, and confirm the routing changes without a redeploy. Weights that
track the config and a ruleset that can move routing without a code change are the two halves
of L7 routing actually working. The full runnable version — Envoy config plus a FastAPI
dispatcher, with a `verify.sh` that checks all of it — is in
[`examples/22-l7-routing/`](https://github.com/patterncatalyst/cloud-native-design-patterns/tree/main/examples/22-l7-routing/)
([README](https://github.com/patterncatalyst/cloud-native-design-patterns/blob/main/examples/22-l7-routing/README.md)).

---

**Source project:** this post is adapted from Appendix I of
[Cloud-Native Design Patterns]({{ page.canonical_project.url }}), where it's part of a
larger tutorial covering event-driven architecture, sagas, observability, and the rest of
the cloud-native API toolbox.
