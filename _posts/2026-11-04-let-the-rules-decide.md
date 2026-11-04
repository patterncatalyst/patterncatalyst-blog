---
title: "Let the Rules Decide: A Camel Content-Based Router on Drools"
date: 2026-11-04
author: "Pattern Catalyst"
tags: [camel, drools, eip, business-rules, java]
categories: [integration]
canonical_project:
  name: "Enterprise Integration Patterns with Apache Camel"
  repo: "patterncatalyst/enterprise-integration-patterns-with-camel"
  url: "https://patterncatalyst.github.io/enterprise-integration-patterns-with-camel/"
excerpt: "A Content-Based Router written as choice().when(...) bakes business policy into a RouteBuilder, so changing a pricing threshold means a redeploy. Hand the decision to a rule engine instead: Camel keeps the fixed topology, Drools owns the when, and toD() is the seam between them."
---

The Content-Based Router is one of the first patterns anyone writing Camel
reaches for, and the first way everyone writes it is a `choice()` block:
`when(header("amount").isGreaterThan(500))` to the express queue, `when(...)` to
fraud review, `otherwise()` to standard. It works, and for routing that never
changes it is the right amount of machinery. The trouble starts when the
branches encode business policy that does change. A pricing tier, a fraud
threshold, a sanctioned-country list: every one of those lives inside a compiled
`RouteBuilder`, so moving the express cutoff from 500 to 750 is a code change, a
build, and a deploy, for a number a business analyst could have told you over
coffee.

The decision and the plumbing are two different concerns wearing one class. Camel
is good at the plumbing: consume from Kafka, unmarshal, dispatch to a
destination, publish. A rule engine is built for the other half: express policy
as data, evaluate it, and let someone who is not a developer change it. The
[Enterprise Integration Patterns with Camel]({{ page.canonical_project.url }})
build pairs them so each does only its own job, with Drools 10 supplying the
decision and Camel carrying it out.

{% include excalidraw.html
   file="drools-camel-router-flow"
   alt="A flow diagram. A Kafka box labelled eip.orders.placed feeds a highlighted Camel route box labelled unmarshal, bean, toD. The Camel route has a two-way arrow to a Drools KIE session box above it, annotated insert fact and read decision, which fires salience rules. From the Camel route, four amber arrows labelled toD of the routing decision fan out to four destination boxes: direct:standard, direct:express, direct:hazmat, and direct:fraud-review. A caption reads: one route, four destinations, decision supplied by the rules."
   caption="Camel owns the topology and the transport; the Drools session supplies the one value that selects a destination" %}

## The route stays dumb

The route does not know the routing policy. It consumes an order, hands the
message to a bean for evaluation, and dispatches on whatever header that bean
set. The branch target is a value rather than a `when()` ladder:

```java
// examples/24-drools-rules/quarkus/.../OrderRoutingRoute.java
from("kafka:eip.orders.placed?brokers={{kafka.brokers}}&groupId=rule-router")
    .routeId("drools-content-router")
    .unmarshal().json(java.util.Map.class)
    .bean("ruleBasedRouter", "evaluate")
    .log("Rule engine decided: ${header.routingDecision} for order ${body[order_id]}")
    .toD("direct:${header.routingDecision}");

from("direct:express")
    .routeId("route-express")
    .marshal().json()
    .to("kafka:eip.orders.express?brokers={{kafka.brokers}}");
// direct:standard, direct:hazmat, direct:fraud-review follow the same shape
```

`toD` is the hinge. Camel's dynamic-to evaluates its expression per message, so
`direct:${header.routingDecision}` resolves to `direct:express`,
`direct:hazmat`, or whichever endpoint the decision named. Adding a fifth
category later is a new `direct:` route and a new rule, with the dispatch line
untouched. The topology is fixed and declarative; the only thing that varies at
runtime is one header value.

## The decision moves to DRL

The policy lives in a rules file, where each rule is an independent
`when`/`then` that a reviewer can read without tracing Java control flow:

```drl
// examples/24-drools-rules/quarkus/.../order-routing.drl
rule "Route hazmat orders to hazmat handler"
    salience 40
when
    $o: Order(containsHazmat == true, routingDecision == null)
then
    $o.setRoutingDecision("hazmat");
    update($o);
end

rule "Route high-value international to fraud review"
    salience 30
when
    $o: Order(amount >= 10000, destinationCountry != "US", routingDecision == null)
then
    $o.setRoutingDecision("fraud-review");
    update($o);
end

rule "Default routing"
    salience 10
when
    $o: Order(routingDecision == null)
then
    $o.setRoutingDecision("standard");
    update($o);
end
```

Three details carry the behavior. `salience` sets priority, so hazmat is tested
before high-value, which is tested before express, down to the default at the
bottom. The `routingDecision == null` guard on every rule means that once one
rule has set a decision, no later rule re-fires, which turns a salience-ordered
rule set into first-match-wins, the same semantics the `choice()`/`otherwise()`
ladder had. And `update($o)` tells the engine the fact changed so the guard is
re-evaluated. Drop the guard and `update`, and every matching rule fires and the
last one wins, which is a different and usually wrong behavior.

## The bean is the bridge

A small CDI bean connects the two worlds. Camel calls it by name from the route;
it builds the facts, runs a session, and writes the result back onto the
exchange:

```java
// examples/24-drools-rules/quarkus/.../RuleBasedRouter.java
@ApplicationScoped
@Named("ruleBasedRouter")
public class RuleBasedRouter {

    private final KieContainer kieContainer;

    public RuleBasedRouter() {
        KieServices ks = KieServices.Factory.get();
        this.kieContainer = ks.getKieClasspathContainer();
    }

    public void evaluate(Exchange exchange) {
        @SuppressWarnings("unchecked")
        Map<String, Object> body = exchange.getIn().getBody(Map.class);

        Order order = new Order();
        order.setOrderId(((Number) body.get("order_id")).longValue());
        order.setAmount(((Number) body.get("amount")).doubleValue());
        order.setDestinationCountry((String) body.getOrDefault("destination_country", "US"));
        order.setContainsHazmat(Boolean.TRUE.equals(body.get("contains_hazmat")));

        KieSession session = kieContainer.newKieSession("orderSession");
        try {
            session.insert(order);
            session.fireAllRules();
        } finally {
            session.dispose();
        }

        exchange.getIn().setHeader("routingDecision", order.getRoutingDecision());
    }
}
```

The session is created, fired, and disposed per message. That keeps the router
stateless: no fact survives from one order to the next, which is what you want
for a stream of independent messages. A `META-INF/kmodule.xml` names the
knowledge base and the session the bean asks for:

```xml
<kmodule xmlns="http://www.drools.org/xsd/kmodule">
    <kbase name="orderRules" packages="com.example.eip.drools">
        <ksession name="orderSession"/>
    </kbase>
</kmodule>
```

There is one idiom here that reads like a style slip.
`Order` is a plain mutable JavaBean with getters and setters rather than a
`record`. A Drools fact has to be mutable, because the rule body calls `setRoutingDecision`
and `update` mutates it in working memory. A record would compile and then fail
to do the one thing the rules need. The mutable POJO is the correct shape for a
fact; leave it alone.

## Decision tables for the people who own the policy

DRL is readable, but it is still a file in the source tree. For policy that
business analysts own outright, Drools reads the same rules from a spreadsheet
decision table, one row per rule:

| Rule Name | Amount Min | Country | Hazmat | Routing Decision |
|-----------|-----------|---------|--------|-----------------|
| Hazmat | any | any | true | hazmat |
| High-value intl | 10000 | != US | any | fraud-review |
| Express | 500 | any | any | express |
| Standard | any | any | any | standard |

The analyst edits a row, the application reloads the table, and the express
cutoff moved without a developer in the loop. That is the whole point of pulling
the decision out of the `RouteBuilder`: the people who own the policy can change
it where they already work.

{% include excalidraw.html
   file="drools-decision-placement"
   alt="Two panels side by side. The left panel, decision baked into Java, shows a dashed box reading choice().when(amount >= 500) inside the RouteBuilder, with an arrow down to a box reading change a threshold means redeploy the service. The right panel, decision externalized to rules, shows a highlighted box reading order-routing.drl or decision table, outside the RouteBuilder, with an amber arrow down to a box reading change a threshold means edit a rule and reload. A caption reads: the router topology never changes, only who owns the when."
   caption="The same router, two places to put the decision: in compiled Java behind a redeploy, or in rules an analyst can reload" %}

## What it buys, and what to watch

Externalizing the decision buys three things a `choice()` block cannot. Policy
changes without a redeploy, through a rule reload or a new decision-table
revision. The rules carry their own audit trail, versioned as data and logged as
they fire, instead of hiding in a git diff of control flow. And ownership moves
to the people who understand the policy, since a salience-ordered rule or a
spreadsheet row reads like the business statement it encodes.

The cost is modest and bounded. There is now a rule engine in the message path,
so you keep the session stateless and per-message, keep facts to plain beans,
and write rule-unit tests the way you would test the `when()` branches they
replaced. For routing that never changes, a `choice()` is still the simpler
option. But the moment the branches start encoding policy that someone
outside the codebase cares about, the right move is to let Camel route and let
the rules decide.

## Source project

This post is mined from
[Appendix F: Drools and Business Rules](https://patterncatalyst.github.io/enterprise-integration-patterns-with-camel/docs/24-appendix-drools/)
of [Enterprise Integration Patterns with Apache Camel]({{ page.canonical_project.url }}),
which covers rule units, decision tables, and the Drools 10 dependency set in
full.

The code shown is the Quarkus example at
[`examples/24-drools-rules/quarkus`](https://github.com/patterncatalyst/enterprise-integration-patterns-with-camel/tree/main/examples/24-drools-rules/quarkus):
the [Camel route](https://github.com/patterncatalyst/enterprise-integration-patterns-with-camel/blob/main/examples/24-drools-rules/quarkus/src/main/java/com/example/eip/drools/OrderRoutingRoute.java),
the [rule-engine bean](https://github.com/patterncatalyst/enterprise-integration-patterns-with-camel/blob/main/examples/24-drools-rules/quarkus/src/main/java/com/example/eip/drools/RuleBasedRouter.java),
and the [DRL rules](https://github.com/patterncatalyst/enterprise-integration-patterns-with-camel/blob/main/examples/24-drools-rules/quarkus/src/main/resources/com/example/eip/drools/order-routing.drl).
A Spring Boot twin lives alongside it under the same example directory.

Full project: [site]({{ page.canonical_project.url }})
· [repo](https://github.com/patterncatalyst/enterprise-integration-patterns-with-camel).
More field notes on the [Pattern Catalyst hub]({{ site.hub_url }}).
