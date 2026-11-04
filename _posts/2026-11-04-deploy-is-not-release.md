---
title: "Deploy Is Not Release: Runtime Route Control with Camel and Feature Flags"
date: 2026-11-04
author: "Pattern Catalyst"
tags: [camel, feature-flags, openfeature, progressive-delivery, quarkus]
categories: [integration]
canonical_project:
  name: "Enterprise Integration Patterns with Apache Camel"
  repo: "patterncatalyst/enterprise-integration-patterns-with-camel"
  url: "https://patterncatalyst.github.io/enterprise-integration-patterns-with-camel/"
excerpt: "Deploying an artifact and releasing a feature are different events, and a flag is what splits them. In a Quarkus + Camel route that means the route ships once, dark, and the rollout, the A/B test, and the targeted enablement are all flag flips evaluated per message, with no redeploy on either the release or the rollback."
---

Progressive delivery rests on one reframe: deploying an artifact
and releasing a feature are different events. The
[Cloud-Native Design Patterns](https://patterncatalyst.github.io/cloud-native-design-patterns/docs/27-appendix-n-feature-flags/) treatment of feature flags
puts it this way: once the code is in production, a flag decides per message
whether the new path is taken, so you deploy continuously and dark, then release
gradually and reversibly, with no redeploy on the critical path of either the
rollout or the rollback.

A Camel route applies that directly. The route ships once, with every
variant of its behavior compiled in and turned off. After that, rolling a new
enrichment step out to 10% of traffic, A/B testing two routing algorithms, or
enabling a compliance change for enterprise customers only are all the same kind
of action: edit a flag definition, and the next message through the route takes
the new path. The
[Enterprise Integration Patterns with Camel]({{ page.canonical_project.url }})
build wires this up with OpenFeature and flagd on Quarkus, and the route becomes
a control surface rather than a thing you redeploy to change.

## The flag is the switch, the default is the contract

A small CDI bean wraps the OpenFeature client and goes into the Camel registry
as `featureFlags`, so any route can ask it for a flag value. The design decision
that matters is in how it fails:

```java
// examples/26-feature-flags/quarkus/.../FeatureFlagEvaluator.java
@ApplicationScoped
@Named("featureFlags")
public class FeatureFlagEvaluator {

    @PostConstruct
    void init() {
        OpenFeatureAPI api = OpenFeatureAPI.getInstance();
        FlagdOptions options = FlagdOptions.builder().host(flagdHost).port(flagdPort).build();
        // setProvider, not setProviderAndWait: the provider connects to flagd in
        // the background. Every lookup below carries a default, so an evaluation
        // made before the stream is up gets the default rather than failing.
        api.setProvider(new FlagdProvider(options));
        client = api.getClient("eip-shipping");
    }

    /** Boolean flag, defaulting to false when flagd cannot be reached. */
    public boolean isEnabled(String flagKey) {
        return client.getBooleanValue(flagKey, false);
    }

    /** String variant with evaluation context, for targeting and fractional rollout. */
    public String getVariant(String flagKey, String fallback, Map<String, Object> context) {
        MutableContext evalContext = new MutableContext();
        context.forEach((k, v) -> evalContext.add(k, String.valueOf(v)));
        return client.getStringValue(flagKey, fallback, evalContext);
    }
}
```

Every lookup takes a default. That is the contract, not defensive boilerplate: if
flagd is unreachable, the SDK returns the default and the route keeps running on
its old behavior. A feature-flag system that can take your application down with
it is worse than no feature-flag system. Using `setProvider` rather than
`setProviderAndWait` is the same decision at startup, since blocking the boot on
the flag stream only turns a slow flagd into a failed launch.

{% include excalidraw.html
   file="flagd-camel-evaluation"
   alt="A Kafka order-placed box feeds a highlighted Camel route box labelled toD by variant, which has a two-way lookup-and-result arrow to a featureFlags bean box labelled OpenFeature client. From the bean, an amber arrow labelled evaluate goes to a flagd daemon box that reads flags.json, and a dashed arrow labelled fallback goes to a greyed default value box labelled flagd unreachable. Captions read: a flag lookup per message with a default that cannot fail, and if flagd is down the route keeps its old behavior."
   caption="The route asks the flag bean per message; the answer comes from flagd, or from the default when flagd is unreachable" %}

## Gradual rollout: a percentage in a file

flagd reads flag definitions from a file it watches, so a rollout is a number in
JSON. The `new-routing-algorithm` flag splits traffic ninety-ten between two
variants:

```json
// examples/_infra/flagd/flags.json
"new-routing-algorithm": {
  "state": "ENABLED",
  "variants": { "legacy": "content-based-router", "new": "dynamic-router" },
  "defaultVariant": "legacy",
  "targeting": {
    "fractional": [ ["legacy", 90], ["new", 10] ]
  }
}
```

The route asks for the variant and dispatches to it with a dynamic-to. The one
subtlety is the targeting key: pass the order id, and flagd hashes it to pick a
side, so a given order always lands on the same variant instead of flipping
per call:

```java
// examples/26-feature-flags/quarkus/.../FeatureFlagRoutes.java
from("kafka:eip.orders.placed?brokers={{kafka.brokers}}&groupId=flag-ab-test&autoOffsetReset=earliest")
    .routeId("feature-flag-ab-test")
    .unmarshal().json(Map.class)
    .process(exchange -> {
        @SuppressWarnings("unchecked")
        Map<String, Object> order = exchange.getIn().getBody(Map.class);
        String algorithm = flags.getVariant(
            "new-routing-algorithm", "content-based-router",
            Map.of("targetingKey", String.valueOf(order.get("order_id")),
                   "customer_tier", String.valueOf(order.getOrDefault("customer_tier", "STANDARD"))));
        exchange.getIn().setHeader("routingAlgorithm", algorithm);
    })
    // The variant string comes from outside the application, so constrain
    // what toD is allowed to resolve it to.
    .toD().allowedSchemes("direct").uri("direct:${header.routingAlgorithm}");
```

To ramp from 10% to 50%, edit the two numbers and save. No rebuild, no restart,
no redeploy, and the split stays sticky across the change because the hash of the
order id is stable. That stickiness is what makes the A/B test valid: the same
order is never counted on both sides.

## A/B and targeting: same flag, different answer per message

The same machinery does cohort targeting. `hazmat-compliance-v2` is on only for
enterprise and VIP customers, decided by the evaluation context rather than by
anything in the route:

```json
// examples/_infra/flagd/flags.json
"hazmat-compliance-v2": {
  "state": "ENABLED",
  "variants": { "v1": "v1", "v2": "v2" },
  "defaultVariant": "v1",
  "targeting": {
    "if": [
      { "in": [ { "var": "customer_tier" }, ["ENTERPRISE", "VIP"] ] },
      "v2",
      "v1"
    ]
  }
}
```

The route passes `customer_tier` in the context and dispatches on the variant it
gets back. An enterprise order routes to `direct:orders-hazmat-v2`; a standard
order to `direct:orders-hazmat-v1`, from the same route, on the same deployment.
Who gets the new compliance path is a property of the flag, and widening the
cohort to a third tier is one more entry in the targeting rule.

## Runtime route control, and the guardrails that make it safe

The through-line is that the route's behavior is controlled from outside the
artifact. Deploy ships the code with the new paths dark; release is a flag flip;
rollback is narrowing the same flag. Nothing on the release or rollback path is a
new deployment.

{% include excalidraw.html
   file="deploy-is-not-release"
   alt="A left-to-right sequence of five boxes connected by amber arrows. Deploy dark (flag off), then Release 1% (flag on, cohort), then Ramp the % (10, 50, ...), then GA 100%, then a greyed Retire the flag. The first arrow is labelled flag flip. Captions read: deploy ships the code, release is a flag flip, and so is rollback; and rollback equals narrow the flag, never a new deploy."
   caption="Every step after the deploy is a change to the flag file, reversible by narrowing it" %}

Handing a route's behavior to an external file needs guardrails, and the example
builds in three. The default on every lookup makes flagd's absence fail static
to the old behavior instead of taking the route down. The targeting key keeps a
rollout sticky, so an order does not change variant mid-stream. And
`toD().allowedSchemes("direct")` constrains the dynamic endpoint: because the
variant string arrives from outside the application, the route pins what it is
allowed to resolve to, closing the same dynamic-URI hole that any content-based
`toD` has to think about.

One operational note worth keeping: pin the flagd image and pin it recent. An
older flagd against this Java provider connects for a hand-run evaluation but
never reaches ready on its event stream, so the SDK returns the default
for every flag and the application looks like it is ignoring the flag file
entirely. The failure hides behind the same default that keeps you safe, so the
version pin is part of the safety story, not separate from it.

## Source project

This post is mined from
[Appendix H: Feature Flags](https://patterncatalyst.github.io/enterprise-integration-patterns-with-camel/docs/26-appendix-feature-flags/)
of [Enterprise Integration Patterns with Apache Camel]({{ page.canonical_project.url }}),
with its progressive-delivery framing drawn from
[Feature Flags & Progressive Delivery](https://patterncatalyst.github.io/cloud-native-design-patterns/docs/27-appendix-n-feature-flags/)
in [Cloud-Native Design Patterns](https://patterncatalyst.github.io/cloud-native-design-patterns/).

The code shown is the Quarkus example at
[`examples/26-feature-flags/quarkus`](https://github.com/patterncatalyst/enterprise-integration-patterns-with-camel/tree/main/examples/26-feature-flags/quarkus):
the [flag evaluator](https://github.com/patterncatalyst/enterprise-integration-patterns-with-camel/blob/main/examples/26-feature-flags/quarkus/src/main/java/com/example/eip/flags/FeatureFlagEvaluator.java),
the [flag-controlled routes](https://github.com/patterncatalyst/enterprise-integration-patterns-with-camel/blob/main/examples/26-feature-flags/quarkus/src/main/java/com/example/eip/flags/FeatureFlagRoutes.java),
and the [flag definitions](https://github.com/patterncatalyst/enterprise-integration-patterns-with-camel/blob/main/examples/_infra/flagd/flags.json).
A Spring Boot twin lives alongside it under the same example directory.

Full projects:
[EIP with Camel]({{ page.canonical_project.url }})
· [Cloud-Native Design Patterns](https://patterncatalyst.github.io/cloud-native-design-patterns/).
More field notes on the [Pattern Catalyst hub]({{ site.hub_url }}).
