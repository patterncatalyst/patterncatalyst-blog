---
title: "Who Knows the Sequence? Three Ways to Coordinate an Order Saga in Quarkus"
date: 2026-10-10
author: "Pattern Catalyst"
tags: [quarkus, kafka, camel, orchestration, event-driven, saga]
categories: [cloud-native]
canonical_project:
  name: "Data Mesh Reference Architecture · Quarkus"
  repo: "patterncatalyst/datamesh-reference-arch-quarkus"
  url: "https://patterncatalyst.github.io/datamesh-reference-arch-quarkus/"
excerpt: "An order places, a payment captures, a shipment dispatches. Who decides that order? Kafka choreography, a Camel route, and a Quarkus Flow document each answer differently, and the real split isn't declarative versus imperative. It's who holds the sequence when production breaks."
---

Every event-driven system eventually runs into one question, and it is not a
framework question. When several steps have to happen in order, who decides the
order? An order is placed, a payment is captured, a shipment is dispatched, a
customer is notified. Something has to know that `payment` comes before
`shipping`. The interesting part is that "something" can live in three
different places, and where you put it decides more about the system than which
library you picked.

The Quarkus data-mesh reference architecture runs all three answers side by side
over the same order-triage domain, with the same business logic underneath, so
the only thing that varies is the coordination shape. That makes it a clean place
to compare them. This post walks the three, then gets to the distinction that
matters in an incident: not whether the sequence is written as code or declared
as data, but whether any single artifact knows the sequence at all.

## One domain, three coordinators

The three legs coordinate the same work. Two of them are "orchestration" and
still disagree sharply, which is the first clue that the usual two-way framing is
too coarse.

- **Kafka choreography.** Four services react to each other through topics with
  no coordinator. `order-service` publishes `order.placed` and has never heard of
  the others. `payment-service` subscribes to `order.placed` and emits
  `payment.captured`; `shipping-service` subscribes to that and emits
  `shipment.dispatched`; `notification-service` also subscribes to `order.placed`
  directly, a fourth reaction running in parallel with the payment chain.
- **A Camel route.** One route in `ai-rules-service` sequences the steps
  imperatively, top to bottom, in a single file.
- **A Quarkus Flow workflow.** The same steps in the same service, declared as a
  task-graph document the engine interprets, structurally the same shape as a
  CNCF Serverless Workflow definition.

{% include excalidraw.html
   file="who-knows-the-sequence"
   alt="Three columns over one order-triage domain. In the Kafka choreography column, four independent services (order, payment, shipping, notification) are joined only by topic-name arrows, with no single box holding the sequence. In the Camel column, one OrderTriageRoute file lists the steps top to bottom, so the sequence lives in code. In the Quarkus Flow column, a workflow document points to two tasks, classify and decide, so the sequence lives in data. The caption under each column answers who knows the sequence: no single artifact, the route, the document."
   caption="One domain, three coordinators. The split that matters is the bottom row: who knows the sequence." %}

## Who knows the sequence

Start with the choreography chain, because its answer is the surprising one: **no
file in the repo contains the sequence.** No artifact holds the string
"order.placed, then payment.captured, then shipment.dispatched" as a single
thing. `PaymentProcessor.process` knows it consumes one topic and produces
another, and nothing more. `ShipmentProcessor.process` knows the mirror image.
Each method is a complete description of what *it* does. That these methods chain
into a three-hop saga is a fact about the running system, not a fact recorded
anywhere in it. Reconstructing "what happens when an order is placed" means
reading four files in four Maven modules and joining them in your head by topic
name. The system has that knowledge; no single part of it does.

The Camel route is the opposite extreme. The route *is* the sequence, and it is
short enough to read in one sitting:

```java
// Camel orchestration — OrderTriageRoute.java
// The route is the sequence: read top to bottom, nothing elided.
from("direct:triage")
    .routeId("triage-order")
    .unmarshal().json(JsonLibrary.Jackson, OrderCreate.class)
    .bean(triageService, "classify")
    .bean(triageService, "decide")
    .marshal().json(JsonLibrary.Jackson);
```

A reviewer who reads those lines has read the entire business process for that
endpoint, in order. The Flow workflow occupies the same slot but answers "what is
the sequence" with a document instead of a trace:

```java
// Quarkus Flow orchestration — OrderTriageWorkflow.java
// Same two steps, declared as a task graph the engine interprets.
return FlowWorkflowBuilder.workflow("order-triage")
    .tasks(
        FlowDSL.function("classify", triageService::classify),
        FlowDSL.function("decide", triageService::decide))
    .build();
```

Both orchestration legs beat choreography on "where do I even look," but they
beat it in different senses. The Camel route's sequence is knowable by *reading
code*. The Flow workflow's sequence is knowable by *reading data*: you can
inspect, diff, or version the descriptor without executing it. That is the
distinction the usual declarative-versus-imperative framing points at, and it is
real. It is just not the one that hurts most at three in the morning.

## Coupling: what changes when you add a step

Measure coupling by asking what has to change, and where, when a step is added.

Add a fifth reaction to `order.placed`, say an analytics service, and the
choreography chain costs the existing services nothing. `order-service` does not
import a client, does not add a call, does not even learn the new service exists.
The new service pays the entire wiring cost by subscribing on its own, exactly as
`notification-service` already does:

```java
// Kafka choreography — PaymentProcessor.java
// Reacts to order.placed; knows nothing of shipping or notification downstream.
@Incoming(Topics.ORDER_PLACED_CHANNEL)
@Outgoing(Topics.PAYMENT_CAPTURED_CHANNEL)
public PaymentCaptured process(OrderPlaced orderPlaced) {
    String orderId = orderPlaced.getOrderId();
    PaymentCaptured existing = paymentStore.findByOrderId(orderId);
    if (existing != null) {
        return existing; // idempotent on redelivery, not a compensation
    }
    // ... build, persist, and return PaymentCaptured ...
}
```

The only shared thing is the schema, the `OrderPlaced` Avro type. No participant
holds a reference to another participant's class, bean, or address.

Here is the part that cuts against the tidy story. Both orchestration legs hold a
direct `@Inject TriageService` reference and call its methods, so their coupling
to the business logic is equally tight, and tighter than choreography's. Adding a
step means editing the one coordinator: in Camel you insert a line into the fluent
chain; in Flow you add an entry to the `.tasks(...)` list. Both are one edit, one
file, one diff, one deploy. On this axis Camel and Flow sit almost on top of each
other, and both sit far from choreography, the declarative-versus-imperative
difference notwithstanding.

## Failure handling: what is implemented, not what is possible

This is where comparisons usually go wrong, because the engines can all do far
more than any given codebase exercises. So be precise about what this repo
implements today.

Choreography's failure handling is **per-hop idempotency, not compensation.**
`PaymentProcessor` and `ShipmentProcessor` both guard against redelivery by
checking for an existing record before acting, which is what makes at-least-once
Kafka delivery safe to build on. But nothing reacts to a *failed* payment by
un-dispatching a shipment, and nothing refunds a capture when a shipment fails.
There is no saga orchestrator and no dead-letter configuration. Kafka's consumer
redelivery is the only safety net wired up.

Camel's failure handling here is **default propagation.** The route has no
`.onException(...)` and no custom error handler, so an exception from `classify`
propagates back through the REST binding. What the route does get by default is
that any exception stops execution at the point of failure, so `decide` never
runs against a `classify` that returned garbage without raising.

Flow's failure handling is, in this repo, **the same as Camel's: unexercised.**
The descriptor declares two tasks and nothing else, no retry policy and no
compensation task. A 120-second timeout exists at the caller boundary, but that
is HTTP hygiene, not workflow-level compensation.

The fair comparison, then, is not "which engine handles failure better." All
three lean on idempotency or exception propagation today. The difference is
**where the fix would go** when you need real compensation:

- In choreography, a new consumer that reacts to a failure event, itself
  published as an event, keeping the pattern decentralized.
- In Camel, `.onException()` or `.doTry()`/`.doCatch()` clauses on the one route.
- In Flow, retry and compensation nodes added to the one workflow document. The
  Serverless Workflow spec supports them as first-class constructs, which is the
  one real edge of the declarative shape: compensation becomes expressible *as
  data*. "The spec supports it" is not the same as "this workflow uses it," and
  keeping those two sentences apart is the difference between reading a demo and
  believing its marketing.

## Debuggability

An incident forces a concrete question: where do you put your eyes first?

For choreography, the answer is "it depends which hop is silent," and the path is
per-service. Check `order-service`'s logs and the `order.placed` topic, then
`payment-service` and `payment.captured`, then `shipping-service`. There is no
single place that reports "did the whole chain finish." That is the standing cost
of loose coupling, paid on every investigation.

The Camel route is one log stream: one request, one thread, one place to read.
The Flow workflow is inspectable per instance and per task, because progress is
tracked against named nodes rather than only as a log stream. That is a real
advantage for long-running or branching workflows. For this particular two-task,
caller-blocks-on-await workflow it stays mostly theoretical, a caveat the
comparison should name rather than skip.

## Operational cost tracks coupling, almost exactly

Running choreography in production means running, monitoring, scaling, and
deploying four services plus Kafka plus a schema registry. The payoff is that
those failure domains are independent: a slow `notification-service` never backs
up `payment-service`, because they share nothing but a topic. Both orchestration
legs run inside a *single* service, so they are cheaper to operate and they share
fate. If `ai-rules-service` is down, both endpoints are down together; if the
shared Ollama model is slow, both feel it identically, because they call through
the same bean.

{% include excalidraw.html
   file="coupling-and-operational-cost"
   alt="Three side-by-side columns comparing the same order-triage saga under three coordinators. The Kafka choreography column: no coordinator, who-knows-it is no single artifact (four handlers joined by topic), coupling is loosest (shared schema only), ops cost is four services plus Kafka plus a schema registry, failure today is per-hop idempotency with no cross-hop compensation; best fit is an event others react to. The Camel route column: sequence in code, who-knows-it is one route read top to bottom, coupling is tight to the business bean (one file to edit), ops cost is one service with shared fate, failure today is default exception propagation; best fit is the sequence as reviewable code. The Quarkus Flow column: sequence in data, who-knows-it is one document (a declared task graph), coupling is tight to the business bean (one file to edit), ops cost is one service with shared fate, failure today is caller-side timeout only; best fit is an inspectable, versionable document."
   caption="One saga, three coordinators, compared on the axes that bite in production: who knows the sequence, coupling, operational cost, and failure handling today." %}

So the trade is legible. Choreography's loosest coupling buys the most
independent operability at the highest service count. Both orchestration legs buy
operational simplicity at the cost of shared fate for everything the coordinator
touches.

## Reach for which, when

| | Kafka choreography | Camel route | Quarkus Flow |
|---|---|---|---|
| Who knows the sequence | No one artifact; reconstructed from four handlers joined by topic name | One route, read top to bottom | One document, read as a declared task graph |
| Coupling | Loosest; shared schema only | Tight to the business bean; one file to edit | Tight to the business bean; one file to edit |
| Failure handling today | Per-hop idempotency; no cross-hop compensation | Default exception propagation | Caller-side timeout only |
| Where compensation would go | A new consumer for a failure event | `.onException()` on the route | Retry/compensation nodes in the document |
| Debuggability | Per-service logs and per-topic inspection | One log stream, one thread | Per-instance, per-task history |
| Operational cost | Four services + Kafka + registry | One service, shared fate | One service, shared fate |
| Best fit | An event other parts of the system react to, where the publisher should not know who is listening | A fixed process where the sequence is the valuable, reviewable artifact and you want imperative control | A fixed process where you want the sequence as an inspectable, versionable document and the engine's retry/compensation vocabulary on hand |

## The question worth carrying

The split that survives contact with production is not declarative versus
imperative. Camel and Flow land in nearly the same place on coupling, on where
the business logic lives, and on how much failure handling exists today. They
differ in how you would extend each one: edit a method chain, or edit a task
graph. Choreography is the real outlier, because it removes the central artifact
entirely, and that single decision ripples through coupling, debuggability,
testing, and operational cost in lockstep.

So when you reach for a coordinator, ask the question the three columns answer
differently. When this breaks in production, who knows the sequence? If the
answer is "no one, by design, and that is the point," you want choreography. If
the answer needs to be "this file," you want a route. If it needs to be "this
document, which we can inspect and version," you want a workflow. Pick the shape
whose answer you can live with during an incident, and the library choice mostly
follows from there.

## Bonus: Camel's Saga EIP, and why it changes the state story

Here is a property the three coordinators share: all of them are
stateless. The Camel route and the Flow document each *describe* the sequence,
but the running service holds nothing between requests. A request arrives, the
sequence executes, the service forgets. That is what lets you run many replicas
and scale them to zero without a second thought, and it is the same posture a
well-behaved service keeps in any language.

Camel has a first-class way to make a route survive partial failure: the Saga
EIP, its implementation of the saga / long-running-action pattern. You mark the
route `.saga()`, and each participating step registers a `.compensation()`
endpoint (and, if it needs one, a `.completion()` endpoint). If a later step
fails, Camel calls the earlier steps' compensations in reverse order to undo
their side effects:

```java
// Camel Saga — OrderSagaRoute.java
// Same top-to-bottom sequence, but each step now declares how to undo itself.
from("direct:place-order")
    .saga()
        .timeout(30, TimeUnit.SECONDS)           // a stuck saga auto-compensates
    .to("direct:capture-payment")
    .to("direct:arrange-shipping");

from("direct:capture-payment")
    .saga()
        .propagation(SagaPropagation.MANDATORY)
        .compensation("direct:refund-payment")   // called in reverse if a later step fails
        .option("paymentId", simple("${body.paymentId}"))
    .bean(paymentService, "capture");

from("direct:refund-payment")                     // the compensating action — must be idempotent
    .bean(paymentService, "refund");              // reads the paymentId saved via option()
```

That is a real improvement over the default exception propagation in the
comparison above. But look at what it costs. A saga is a state machine: it moves
through Active, Completing, and Completed on the happy path, or through
Compensating and Compensated on failure, with a timeout that auto-compensates a
saga that gets stuck. Something has to hold where each in-flight saga currently
is.

{% include excalidraw.html
   file="camel-saga-state-machine"
   alt="A state machine on the left and a list of persisted state on the right. The state machine starts at Active (saga started). On all steps ok it moves to Completing (run completion URIs) and then Completed. On error or timeout it moves from Active to Compensating (undo in reverse) and then, via the compensation URIs, to Compensated. The right panel lists what the coordinator must persist per in-flight saga: the participants and their compensation and completion URIs, the saved option() values such as the ids a refund needs, the status and which step is next, and the timeout deadline. A note explains that InMemorySagaService keeps this in the JVM heap so a crash mid-saga loses it and nothing compensates, whereas an LRA coordinator persists it and the Camel service goes back to stateless."
   caption="A saga is a state machine. The sequence can live in code; the saga's progress has to live in a store." %}

The default `CamelSagaService` is `InMemorySagaService`, and it keeps all of
that in the JVM heap. Fine for a demo, and fatal in production: if the process
dies between "payment captured" and "shipping arranged," the saga state dies with
it, no compensation ever runs, and you are left with a payment that was captured
and will never be refunded. The stateless property we just praised is gone, in
the one way that hurts most.

So making a Camel saga correct is, in the end, a state-externalization exercise.
You delegate the saga state to a coordinator that persists it: `camel-lra` plus a
MicroProfile LRA coordinator (Narayana) writes the participant list, the
compensation URIs, and the saved options to durable storage, and after a restart
the coordinator drives the outstanding compensations to completion.

```java
// Non-durable: saga state lives in the JVM heap, lost on crash.
camelContext.addService(new InMemorySagaService());

// Durable: delegate saga state to an LRA coordinator that persists it,
// so the Camel service itself goes back to holding nothing.
LRASagaService saga = new LRASagaService();
saga.setCoordinatorUrl("http://lra-coordinator:8080");
saga.setLocalParticipantUrl("http://ai-rules-service:8080");
camelContext.addService(saga);
```

The rest of the work follows from treating it as a state machine with external
storage: give every step a compensation route, use `.option(...)` to persist the
identifiers those compensations need, make each compensation idempotent because
the coordinator may retry it, and set a `.timeout(...)` so a wedged saga does not
sit half-applied forever. Do that and the state lives outside the process again,
and the service is stateless once more. It is the same externalize-your-state
move that keeps any service replaceable, applied to the one piece of coordination
that does need to remember something.

This is why the saga sits in a bonus rather than in the three-way comparison. The
route and the workflow answer "who knows the sequence." The saga answers a second
question on top of it, "who knows how far we got, and will that knowledge survive
a crash," and answering it well means putting the progress somewhere durable. The
sequence can live in code or in a document. The saga's state has to live in a
store. Keeping those two questions separate is the whole point: add a saga only
when you need the second answer, and know that you are trading the stateless
posture for a coordinator you now have to operate.

For the mechanics, see Camel's
[Saga EIP](https://camel.apache.org/components/next/eips/saga-eip.html) and the
[camel-lra component](https://camel.apache.org/components/next/others/lra.html).

---

**Source project.** This post is drawn from
[Chapter 13, orchestration styles](https://patterncatalyst.github.io/datamesh-reference-arch-quarkus/docs/13-orchestration-styles/)
and the deeper
[Appendix comparing the three engines](https://patterncatalyst.github.io/datamesh-reference-arch-quarkus/docs/21-orchestration-engines-compared/)
of the
[Data Mesh Reference Architecture on Quarkus](https://patterncatalyst.github.io/datamesh-reference-arch-quarkus/),
which runs all three legs green over one shipping/order domain. The code shown
here is real:
[`OrderTriageRoute.java`](https://github.com/patterncatalyst/datamesh-reference-arch-quarkus/blob/main/examples/ai-rules-service/src/main/java/com/patterncatalyst/datamesh/airules/OrderTriageRoute.java),
[`OrderTriageWorkflow.java`](https://github.com/patterncatalyst/datamesh-reference-arch-quarkus/blob/main/examples/ai-rules-service/src/main/java/com/patterncatalyst/datamesh/airules/OrderTriageWorkflow.java),
and
[`PaymentProcessor.java`](https://github.com/patterncatalyst/datamesh-reference-arch-quarkus/blob/main/examples/payment-service/src/main/java/com/patterncatalyst/datamesh/payment/PaymentProcessor.java).
Full source is on
[GitHub](https://github.com/patterncatalyst/datamesh-reference-arch-quarkus).
