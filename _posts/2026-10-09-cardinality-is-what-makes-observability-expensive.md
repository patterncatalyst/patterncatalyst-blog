---
title: "Cardinality, Not Instrumentation, Is What Makes Observability Expensive"
date: 2026-10-09
author: "Pattern Catalyst"
tags: [observability, opentelemetry, cardinality, cost, metrics]
categories: [observability]
canonical_project:
  name: "OpenTelemetry Observability Tutorial"
  repo: "patterncatalyst/otel-observability-tutorial"
  url: "https://patterncatalyst.github.io/otel-observability-tutorial/"
excerpt: "A metric isn't one number, it's a family of time series, one per unique label combination. Add a single unbounded label and a cheap counter becomes a six-figure line item. Here's how cardinality drives observability cost, and where to control it."
---

A dashboard that runs fine in a demo can start timing out, or show up on a
billing report as a six-figure line item, the first time it meets production
traffic. The instrumentation code did not change. The volume and the shape of
what it emits did. Cardinality is the single biggest lever on observability
cost, and it is almost always invisible at the point where the damage is done:
a line of code that looked harmless when it was written.

Three cost problems hide behind that bill, and they are distinct: metric
cardinality, log volume, and trace volume. Each has its own failure mode and
its own fix, and all three get cheaper to control the earlier you give the
collector a say in what leaves the building.

## A metric is a family of time series

Start with the part that surprises people most. A metric is not one stream of
numbers. It is a family of time series, one for every unique combination of
label values it has ever carried. A counter named `http_requests_total` with
labels `method`, `route`, and `status_code` does not produce one series; it
produces one per observed combination: `GET /cart 200`, `POST /checkout 500`,
`GET /cart 404`, and so on. The metrics backend (Mimir, in the stack this is
drawn from) allocates storage, an index entry, and query-time bookkeeping for
every one of them, and keeps it until retention ages it out.

That is fine when the label values come from a small, closed set. HTTP method
has a handful of values. A templated route such as `/cart/{id}` has a few
dozen. Status code has a few hundred, of which a service exercises maybe
twenty. Multiply those together and a well-behaved metric produces a few
thousand series, which any backend built in the last decade handles without
noticing.

The failure mode is a label whose value set has no ceiling: a raw user ID, a
session token, a request ID, an email address, a full URL with its query string
still attached, a container ID. Each one multiplies into the series count, and
because the set of values grows without bound, the series count grows without
bound too. A counter that looked like it tracked requests per route quietly
becomes requests per route per user who has ever hit the service, and the
backend ends up carrying a series for every person who tried the product once
two years ago and never came back. That is a cardinality explosion, and it is
the most common reason a metrics backend falls over or a bill spikes.

{% include excalidraw.html
   file="cardinality-explosion"
   alt="Diagram comparing two versions of the same counter http_requests_total. With a bounded label set of method, route, and status it produces about three thousand time series, with cheap storage and fast queries. Adding one unbounded label, user_id, turns the same counter into millions of time series, with high storage cost and slow dashboards. The metric definition looks nearly identical in both cases."
   caption="One unbounded label turns a few thousand series into millions; the code looks the same at review time" %}

OpenTelemetry's own guidance is explicit here: attributes with unbounded value
sets belong on spans and logs, which carry per-event detail, not on metric
labels, which exist to be aggregated (see the
[semantic conventions](https://opentelemetry.io/docs/specs/semconv/) and the
[metrics specification](https://opentelemetry.io/docs/specs/otel/metrics/)). The
rule of thumb that holds up across every language: before adding a label to a
metric, ask whether you could list every value it will ever take. If you
cannot, it does not belong on that metric.

Cardinality has a second cost that bills as latency rather than dollars. A query
for the error rate of one route has to scan and aggregate every series that
matches, so the more series exist, the more work the engine does to collapse
them into the single line a dashboard shows. An explosion does not only make
storage expensive; it makes every dashboard built on the affected metric
slower, sometimes slow enough to time out. That timeout is usually the first
symptom an on-call engineer notices, long before anyone opens a billing report.

## Label hygiene is a design discipline

Fixing cardinality after the fact is a scramble: finding which series exploded,
patching instrumentation in a dozen places, then waiting out a retention window
for the damage to roll off storage. Applied up front it is cheap, and it comes
down to a few habits.

- **Favor templated routes over literal paths.** A framework that already knows
  the route template should report `/orders/{orderId}`, not `/orders/8f21ac`.
  The identifier belongs on the span or the log line, where it tags one event
  instead of multiplying a whole family of series.
- **Bucket instead of enumerate.** A wide-ranging numeric value, a payload size
  or a duration, belongs in a histogram, whose buckets are a bounded dimension.
  The raw value as a label is not.
- **Push high-cardinality identifiers down the signal stack.** A user ID, order
  ID, or trace ID is exactly the right thing to attach to a span attribute or a
  structured log field. It should never also become a metric label.
- **Review new labels like schema changes.** A reviewer who asks one question,
  "what is the set of values this can take, and is it bounded?", catches almost
  every cardinality problem before it ships, because the answer is usually
  obvious once it is said out loud.

## Logs and traces fail differently

Logs do not explode in the metrics sense, because a log line does not
pre-aggregate into a time series. Loki, the log store in this stack, indexes a
small set of labels and keeps the rest of each line as compressed text. The log
cost problem is blunter: volume. Every `DEBUG` line left on in production, every
health check logged at `INFO`, every framework that logs once per database
query, scales linearly with traffic, and storage is priced by the gigabyte
ingested and the days retained.

Three controls compound here. Log-level discipline keeps `DEBUG` and `TRACE` in
development and short diagnostic windows, not steady-state production.
Deduplication folds a retry loop that would log the same warning on every
attempt into one line with a count. Retention tiering keeps recent data hot and
moves or drops the rest; that is where the multi-month savings come from, more
than trimming what gets logged in the first place.

One cardinality mistake does carry over to logs: a stream label that varies per
request shatters a single log stream into thousands of tiny ones and defeats the
compression that makes Loki affordable. Keep stream labels bounded (service,
environment, level) and let high-cardinality detail live in the line content,
searchable with a query-time filter.

Traces carry the highest per-unit cost of the three, because one trace is a tree
of spans, each with its own attributes, events, and timing, and Tempo stores and
indexes all of it. Capturing every trace is viable at modest traffic and
untenable as volume grows, which is what
[sampling](https://patterncatalyst.github.io/otel-observability-tutorial/docs/20-sampling/)
is for: head and tail strategies decide which traces are worth keeping before
they reach storage. Sampling sits upstream of everything else here, so a service
emitting one hundred percent of its traces is the top candidate for runaway
storage regardless of how clean its spans are. On top of sampling, span
attribute hygiene matters: a span that attaches a full request body, or a stack
trace on every exception, adds bytes to every stored span, multiplied by however
many survive sampling.

## Control spend at the collector

Everything above is something a service owner sets at the instrumentation layer:
which labels to add, which log level to run, which attributes to attach. The
[collector](https://patterncatalyst.github.io/otel-observability-tutorial/docs/19-the-collector/)
is the second, and often more practical, place to enforce it, because it applies
the decision centrally without redeploying every service that emits telemetry.

A `filter` processor drops whole signals, spans, or log records that match a
condition; high-volume, low-value health-check traces are the usual first
target. An `attributes` processor in `delete` mode strips a specific attribute
as telemetry passes through, which is the practical backstop for cardinality:
even if a library attaches a raw user ID to every span by default, the collector
removes it before export rather than waiting for every team to patch its code.

```yaml
processors:
  attributes/drop-user-id:
    actions:
      - key: user.id
        action: delete
  filter/drop-health-checks:
    traces:
      span:
        - 'attributes["http.route"] == "/healthz"'
```

A `transform` processor goes further, rewriting a high-cardinality attribute
into a bucketed or templated form instead of dropping it outright, keeping some
diagnostic value while still bounding the cardinality. For metrics, aggregation
(in the collector, or through SDK views it forwards) lets a team decide a
metric needs an application-wide total rather than a per-pod breakdown, cutting
the series count by the number of replicas without losing the number alerting
depends on.

{% include excalidraw.html
   file="ch21-cost-controls"
   alt="Metrics, logs, and traces flow from services into an OpenTelemetry Collector, where filter, aggregate, and attribute-drop processors reduce volume before it reaches backend storage."
   caption="Controlling spend centrally at the collector, across all three signals" %}

The thread through all of these is central ownership. A platform or
observability team sets retention and cost policy once, in the collector
configuration, and it applies across every service, every language, and every
team, whichever ones were careful about cardinality and which were not. That is
the payoff of putting a collector in the data path: a direct SDK-to-backend
pipeline has no interception point, so every cost control becomes a
per-service, per-language exercise instead of one shared policy.

## The habit worth keeping

Cost control in observability is not a single switch. It is the sum of small
decisions: a bounded label, a sane log level, a sampling rate matched to
traffic, a collector processor that catches what instrumentation missed. None of
them is expensive on its own. What is expensive is discovering that all three
signals have been growing unbounded at once, usually the same morning a billing
alert fires or a backend starts rejecting writes under load.

So carry one question forward. When you add any label, attribute, or log field,
ask what values it can take and whether that set has a ceiling. When you read a
collector configuration, check whether it forwards everything it receives or
actively decides what is worth keeping. A pipeline with no filtering, no
attribute processing, and no sampling is not a more complete observability
setup. It is a more expensive one, usually without a matching gain in anything a
human can use while debugging a live incident.

---

**Source project.** This post is drawn from the
[cardinality and cost chapter](https://patterncatalyst.github.io/otel-observability-tutorial/docs/21-cardinality-and-cost/)
of the
[OpenTelemetry Observability Tutorial](https://patterncatalyst.github.io/otel-observability-tutorial/),
which builds all five signals (traces, metrics, logs, baggage, and profiles)
across Spring Boot, Quarkus, and Python on one Grafana LGTM stack. The
[sampling chapter](https://patterncatalyst.github.io/otel-observability-tutorial/docs/20-sampling/)
covers trace volume in depth, and the
[collector chapter](https://patterncatalyst.github.io/otel-observability-tutorial/docs/19-the-collector/)
covers the data-path pipeline these cost controls plug into. Full source is on
[GitHub](https://github.com/patterncatalyst/otel-observability-tutorial).
