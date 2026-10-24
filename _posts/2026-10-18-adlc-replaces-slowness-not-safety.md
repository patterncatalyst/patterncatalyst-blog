---
title: "An ADLC Replaces the SDLC's Slowness, Not Its Safety"
date: 2026-10-18
author: "Pattern Catalyst"
tags: [adlc, sdlc, agentic, ai-development, modernization, verification]
categories: [ai-development]
canonical_project:
  name: "Modernizing Enterprise Applications"
  repo: "patterncatalyst/modernizing-enterprise-applications"
  url: "https://patterncatalyst.github.io/modernizing-enterprise-applications/"
excerpt: "The software lifecycle most teams run is shaped by one fact: writing code is slow relative to the cost of being wrong. Agents break that assumption, and the serial human gates that provided the SDLC's safety go with it. An Agentic Development Lifecycle is what you adopt to rebuild that safety at the new pace."
---

The software development lifecycle most teams run looks the way it does for one
reason, and it's not bureaucracy. Writing code is slow relative to the cost of
being wrong. When a line takes minutes of human attention to produce and hours to
review properly, you economize on attempts: front-load the thinking into
requirements and design, specialize the roles so an analyst, an architect, a
developer, a tester, a reviewer, and an operator each carry one concern, and
insert a code-review stage as cheap insurance against an expensive mistake. Every
one of those stages is a queue with a context switch at both ends. That's why
modernizing a monolith under the traditional SDLC is measured in quarters per
extracted service: not because the code is hard to write, but because it has to
pass six serially-scheduled human gates to be trusted.

A coding agent breaks the assumption the whole shape rests on. It doesn't remove
those stages; it collapses the time each one takes and folds several into one
continuous loop. "Requirements" becomes stating intent in a paragraph. "Design"
and "implementation" merge, because for an agent there's no handoff cost between
designing a function and writing it. "Testing" and "review" become the agent
writing and running tests alongside the code, with a human reading a finished diff
instead of an abstract plan. Weeks of queueing delay become hours of one engineer
working with one model.

Here's the part most "AI makes us faster" pitches skip. The SDLC's slowness was
also where a lot of its safety lived. Six serial human gates are a crude but real
form of error-correction. Collapse the stages for speed without replacing that
error-correction with something else, and you've built a faster way to ship a
wrong answer. The value of an **Agentic Development Lifecycle (ADLC)** isn't the
speed, which an agent gives you the moment it's in the loop. It's putting the
safety back, in a form that keeps up with the new pace.

## Where the safety moves

The "Modernizing Enterprise Applications" book runs a specific ADLC: seven named
phases, and exactly two points where a human must stop the loop and decide.

{% include excalidraw.html
   file="adlc-replaces-slowness-not-safety"
   alt="Two columns. The left column, the SDLC, is a vertical chain of human-owned stages: requirements and design, implement, test and QA, code review (marked as a human gate), and deploy, with an emphasis box reading that six serial human gates take weeks to months and the slowness was the error-correction. The right column, the ADLC, is a shorter loop: Frame (human states intent), Map plus Plan (agent), a first amber human gate for plan approval, Generate plus Verify (agent), a second amber human gate for equivalence sign-off, and Operate plus Reconcile, with an emphasis box reading two human gates plus one cheap automated gate, hours to days."
   caption="The SDLC's six serial human gates become the ADLC's two irreducible ones, plus a cheap automated gate that runs on every candidate change." %}

The seven phases are **Frame** (a human states the outcome and records it),
**Map** (the agent classifies the module and locates the seam), **Plan** (the
agent proposes an approach), **Generate** (it writes the code and tests),
**Verify** (a second, more skeptical pass runs the suite and a security scan),
**Operate** (the change rolls out behind a flag), and **Reconcile** (the ledger is
updated with what happened). The two human gates sit at Plan and Verify,
and they're the whole argument. An agent can produce a plan, but it can't be
trusted to decide that its own plan is the right one to run against a production
system someone else depends on. An agent can run a suite and report green, but it
can't be the one who decides that green is sufficient evidence to retire the code
it just replaced. The first is a business judgment; the second is a sign-off. Both
stay with a person. Everything between them is delegated.

## The cheap gate that makes speed safe

Two human gates aren't enough on their own. If "Verify" meant a human re-reading
every diff, the review bottleneck would come straight back. The ADLC's answer is to
make verification so cheap it runs after every candidate change, not just before a
release. In this modernization, the monolith's own Newman collection, captured
once against the running "before" system, becomes the **behavior-equivalence
suite**, and it's re-run unchanged against every extracted service:

```bash
# demos/demo-equivalence.sh
# This collection (tooling/newman/mea.postman_collection.json) is the project's
# EQUIVALENCE GATE: it was captured against the running Spring Boot monolith and
# is re-run UNCHANGED against every extracted service. A service is "done" only
# when it passes this same suite with no edits to the collection — only the
# target --baseUrl changes.
newman_cli run "${COLLECTION}" \
    --environment "${ENVIRONMENT}" \
    --env-var "baseUrl=${BASE_URL}" \
    --reporters cli
```

A gate that cheap changes what an agent's speed is worth. Because an agent can
produce several candidate approaches to a tricky extraction in the time a human
would take to produce one, the question is whether those extra attempts are an
asset or a liability. Against a scripted suite that re-runs in seconds, every
failed candidate is caught before anyone ships it, so the attempts become free
experiments. Against a human code review, those same extra candidates are extra
review hours nobody has.

{% include excalidraw.html
   file="equivalence-gate-free-experiments"
   alt="Three candidate extractions (A, B, C) produced by the agent flow into a central amber box, the equivalence gate, labeled as the monolith's Newman suite re-run unchanged. From the gate, candidates A and B flow to discarded outcomes drawn as faint boxes, while candidate C flows to a ships outcome. An emphasis box reads that a failed attempt caught at the gate is a free experiment, not a shipped regression."
   caption="A gate cheap enough to run on every candidate turns the agent's many attempts into free experiments. Speed without that gate is just a faster way to ship a regression." %}

This is why modernization is the case that needs the discipline most. A throwaway
prototype can skip from Generate straight to Operate and nobody is worse off if
it's wrong. Strangling pieces off a monolith is the opposite: refactoring under a
hard constraint (the system must keep behaving exactly as it already does)
applied to code a different team wrote, for reasons nobody fully remembers, that
real customers depend on today. The agentic loop's raw speed makes that the *most*
dangerous category of change without a verify loop tight enough to keep up with
it, and the cheapest tight loop is a scripted contract suite, not a human.

## The ledgers replace the heavyweight documents

The SDLC's audit trail (the design doc, the test plan, the traceability matrix)
also does real work: it records why a decision was made and lets a regulated or
contract-bound team prove the work was done, not just claimed. The ADLC keeps that
work but moves it into three append-only files the agent maintains as it goes:
`decisions.md`, `build-plan.md`, and `reconciliation.md`. One row per decision,
written when the decision is made:

```markdown
<!-- _plans/decisions.md -->
| DRQ-013 | Extraction order = rising difficulty: review → notification →
inventory → payment → shipping → order. | Each step forces exactly the next
pattern; monotonic difficulty curve. | accepted |
| DRQ-014 | The monolith's Newman collection is the behavior-equivalence suite,
run unchanged against each extracted service and gating every extraction in CI. |
Makes "preserves behavior" checkable; per-chapter + CI acceptance gate. | accepted |
```

Because the agent that just made the change can also append the row that records
it, the documentation debt that normally accumulates as a separate backlog item
stops accumulating by default. The ledger costs almost nothing to keep current,
which is the only reason it stays current.

## What the delegation looks like

The abstract loop gets concrete fast. Here is the mechanical part of standing up a
new service, the health-check wiring, written by hand for the Spring Boot
monolith:

```java
// Written by hand, reviewed in a pull request, merged after a day's turnaround.
@Component
public class DatabaseReady implements HealthIndicator {
    private final JdbcTemplate jdbc;
    public DatabaseReady(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    @Override public Health health() {
        try {
            jdbc.queryForObject("SELECT 1", Integer.class);
            return Health.up().build();
        } catch (DataAccessException e) {
            return Health.down(e).build();
        }
    }
}
```

And here is the Quarkus equivalent the agent scaffolds during Generate, using the
Spring-compatibility bridge so the lift stays mechanical:

```java
// Scaffolded by the agent in the Generate phase. Trusting it does not rest on a
// second engineer re-reading every line; it rests on the equivalence suite
// running against it unchanged.
@Readiness
public class DatabaseReady implements HealthCheck {
    @Inject PgPool db;

    @Override public HealthCheckResponse call() {
        return db.query("SELECT 1").executeAndAwait() != null
            ? HealthCheckResponse.up("database")
            : HealthCheckResponse.down("database");
    }
}
```

The two do the same thing by design. The agent didn't write anything cleverer
here; it produced a near-direct translation. What changes is that it exists
in minutes rather than a sprint, and that trusting it rests on a gate that re-runs
in seconds rather than a second engineer's free hour. That's the trade the ADLC
makes everywhere: move the human from the things a gate can check to the two
decisions a gate can't.

## The differences, side by side

| | SDLC | ADLC |
|---|---|---|
| Rests on | Writing code is slow; economize on attempts | An agent makes attempts cheap; economize on *un-caught* attempts |
| Roles | Specialized, serial: analyst → architect → dev → tester → reviewer → operator | One engineer guides; the agent carries the phases between the gates |
| Human gates | Six-ish serial stages, each a queue | Two: plan approval, equivalence sign-off |
| Verification | Human code review, before release | A scripted equivalence suite, after every candidate |
| Audit trail | Design doc, test plan, traceability matrix | Append-only `decisions.md` / `build-plan.md` / `reconciliation.md`, agent-maintained |
| Extra attempts are | Extra review hours nobody has | Free experiments, caught before they ship |
| Pace | Quarters per extracted service | Hours to days per step |

## The takeaway

Adopting an ADLC isn't a decision to go faster; the agent already did that, and
going faster without changing anything else is the dangerous part. It's a decision
about where to spend the human judgment you have left once the typing is cheap.
Put it at the two gates a machine can't stand in for, the plan that commits you to
an approach and the sign-off that retires the old code, and make everything between
them lean on a gate cheap enough to run on every attempt. Do that, and the agent's
speed stops being a way to ship regressions faster and becomes a way to run more
experiments safely. Skip it, and you've kept the SDLC's biggest liability, its
capacity to ship a confident wrong answer, while discarding the slowness that used
to catch it.

---

**Source project.** This post is mined from
[Chapter 5, "From SDLC to ADLC"](https://patterncatalyst.github.io/modernizing-enterprise-applications/docs/05-from-sdlc-to-adlc/)
of [Modernizing Enterprise Applications](https://patterncatalyst.github.io/modernizing-enterprise-applications/),
which strangles a Spring Boot monolith into Quarkus + Camel microservices one seam
at a time, each extraction driven through this lifecycle and gated by the
behavior-equivalence suite. The loop is taken apart further in
[Chapter 6, "Agents, Skills & MCP Tools in the Loop"](https://patterncatalyst.github.io/modernizing-enterprise-applications/docs/06-agents-skills-mcp/)
and run end to end on a trivial change in
[Chapter 7, "The ADLC Safety Net"](https://patterncatalyst.github.io/modernizing-enterprise-applications/docs/07-adlc-safety-net/).
The artifacts shown here:
[`demos/demo-equivalence.sh`](https://github.com/patterncatalyst/modernizing-enterprise-applications/blob/main/demos/demo-equivalence.sh),
[`_plans/decisions.md`](https://github.com/patterncatalyst/modernizing-enterprise-applications/blob/main/_plans/decisions.md),
and the Spring-to-Quarkus health-check lift from Chapter 5. Full source is on
[GitHub](https://github.com/patterncatalyst/modernizing-enterprise-applications).
More from the catalog on the [Pattern Catalyst hub]({{ site.hub_url }}).
