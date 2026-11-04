---
title: "Cut One Seam, Prove It Equal: ADLC Modernization in Practice"
date: 2026-11-03
author: "Pattern Catalyst"
tags: [adlc, modernization, strangler-fig, python, verification]
categories: [ai-development]
canonical_project:
  name: "Modernizing Enterprise Applications (Python)"
  repo: "patterncatalyst/modernizing-enterprise-applications-python"
  url: "https://patterncatalyst.github.io/modernizing-enterprise-applications-python/"
excerpt: "A coding agent can draft a service extraction in an afternoon, which makes the old question louder, not quieter: did the behavior change? A strangler-fig migration run on an Agentic Development Lifecycle answers it one seam at a time, with a behavior-equivalence gate over a real socket standing between every extraction and production."
---

A big-bang rewrite is the most expensive way to modernize a monolith, because it
asks you to be right about the whole system once, at the end, when the cost of
being wrong is highest. The strangler-fig pattern trades that single large bet
for a sequence of small reversible ones: stand a router in front of the monolith,
move one bounded context behind a new service, flip traffic to it, and repeat
until nothing routes to the monolith anymore. Each move is small enough to
understand and cheap enough to undo.

A coding agent changes the economics of each move but not its risk. It can draft
an idiomatic replacement for a bounded context in an afternoon, so the slow part
of an extraction stops being the writing. What stays exactly as hard is the
question every cutover turns on: does the new service behave the same as the code
it replaces, on every path a client can take? Running the migration on an
Agentic Development Lifecycle answers that question with a gate the migration
cannot skip. The
[Python modernization build]({{ page.canonical_project.url }}) strangles a Django
monolith into six FastAPI services, and every seam crosses the same gate before
any traffic does.

## The shape of one seam

The first thing that goes in front of the monolith is a thin reverse proxy. It
owns no business logic. It reads the request path, picks a backend from
operator-set configuration, and relays the upstream response through unchanged.
During a cutover window the choice of backend is the whole point:

```python
# examples/01-strangler-proxy/app/main.py
def _select_backend(settings: Settings, path: str) -> str | None:
    """Operator-configured, request-independent target selection.

    Never builds a target from request data (header, query string, path
    segment), only ever from the fixed base URLs in `settings`, which keeps
    this proxy's dynamic-URI seam closed to SSRF: no caller can redirect it
    to an arbitrary host.
    """
    if path.startswith("/api/reviews"):
        return settings.review_base_url
    if path.startswith("/api/notifications"):
        return settings.notification_base_url
    # ... one prefix per bounded context ...
    return None
```

The target set comes from configuration read once at startup, never from the
inbound request, which closes the proxy's dynamic-URI seam to SSRF by
construction. In the reversible window, a per-context flag decides whether
`/api/reviews` goes to the monolith's review module or to the new review service.
The default is the monolith. The flag flips to the new service only after the
extraction has earned it, and flipping back is a configuration change and a
restart rather than a redeploy.

{% include excalidraw.html
   file="strangler-seam-cutover"
   alt="A client calling /api/reviews points to a strangler proxy labelled route by flag. The proxy fans out to two backends: a dashed, greyed box for the monolith review module labelled flag OFF, being retired, and a solid box for the review service on FastAPI labelled flag ON, once the gate is green. A caption reads: flipping back only changes configuration, with no redeploy."
   caption="One path prefix, two backends, one reversible flag: the proxy makes a cutover a configuration decision rather than a deploy" %}

## Rebuilt idiomatically, with the contract held fixed

The replacement is not a line-by-line port of the Django view. It is written the
way a FastAPI service should be, with dependency-injected sessions, Pydantic
schemas, and per-route auth. What it preserves is the contract on the wire, down
to the route shapes and the `Location` header. Here is the monolith's create
path, in Django REST Framework:

```python
# examples/00-monolith/review/views.py
class ReviewListCreateView(APIView):
    authentication_classes = [BasicAuthentication]

    def get_permissions(self):
        if self.request.method == "POST":
            return [IsAuthenticated()]
        return [AllowAny()]           # GET stays open

    def post(self, request):
        command = ReviewCreateSerializer(data=request.data)
        command.is_valid(raise_exception=True)
        data = command.validated_data
        review = services.create_review(**data)
        body = ReviewSerializer(review).data
        return Response(body, status=201,
                        headers={"Location": f"/api/reviews/{review.id}"})
```

And the same endpoint, rebuilt idiomatically in FastAPI, gating only `POST`
behind auth and emitting the identical relative `Location`:

```python
# examples/02-review-service/app/routes.py
@router.post("/api/reviews", response_model=ReviewOut, status_code=201)
async def create_review(
    payload: ReviewCreate,
    response: Response,
    session: AsyncSession = Depends(get_session),
    _authenticated_username: str = Depends(require_basic_auth),
) -> ReviewOut:
    customer = await repository.get_customer_by_id(session, payload.customer_id)
    if customer is None:
        raise ResourceNotFound(f"No customer with id {payload.customer_id}")
    review = await repository.create_review(session, ...)
    # Matches the monolith's relative Location header exactly.
    response.headers["Location"] = f"/api/reviews/{review.id}"
    return ReviewOut.model_validate(review)
```

Two frameworks and two idioms, but one observable contract. The internals
modernize; the behavior a client can see is held fixed, because holding it fixed
is what makes the next step checkable.

## The gate: behavior equivalence over a real socket

An extraction is trusted when a behavior-equivalence suite says the new service
answers like the monolith across the request surface, and a person agrees that
the suite covers the right surface. In this build the automated half is a Newman
collection re-run against a live process on every candidate build, exercising
both backends in both flag states, with a negative check: kill the review-service
process with the flag on, and every request must come back a real `502`, never a
swallowed `200`. An equivalence gate that cannot fail is not asserting anything.

The gate's design is the part to copy, and the Python service's own routing tests
encode it: the proxy must relay a real upstream error verbatim instead of masking
it.

```python
# examples/01-strangler-proxy/tests/test_routing.py
def make_stub_app(name: str):
    """A minimal raw-ASGI stub backend: echoes request shape as JSON.

    Any path ENDING in `/boom` always answers 500, to prove the proxy relays
    a real upstream error status verbatim instead of raising or masking it.
    """
```

Why insist on a live socket rather than a fast in-process mock? Because the
review service kept sharing the monolith's Postgres schema during the cutover
window, so both backends read and write the same rows. A suite that checked only
the HTTP response shape could not tell a request served correctly by the new
service apart from one that fell through to the monolith and returned the same
data anyway. The real-socket `POST` is the one place those two outcomes diverge,
and running there caught a chunked-body bug that an in-process mock transport was
structurally unable to see: Django's WSGI handler treats a chunked request with
no `Content-Length` as a zero-length body, so every proxied `POST` was arriving
empty. A mock never frames a request over the wire, so it never reproduced the
failure.

{% include excalidraw.html
   file="equivalence-gate-real-socket"
   alt="Two source boxes on the left, the monolith on WSGI labelled golden baseline and the review service on FastAPI labelled candidate, both feed into a central box, the behavior-equivalence suite, annotated real socket, both flag states, plus a negative check. From the suite a solid amber arrow leads to a box labelled match: sign off, flip the flag; a dashed arrow leads to a greyed box labelled in-process mock, misses the chunked-body bug. The caption reads: a real socket proves what a mock cannot."
   caption="The equivalence gate compares the candidate against the monolith baseline on the surface where they differ" %}

## Two human gates bracket the automation

The automation does not run unsupervised. The ADLC puts a person at exactly two
points, and nowhere in between. The first is plan approval, before any diff
exists: a human reads the step plan for the extraction, including the decision to
defer schema ownership and keep the shared table for now, and signs off on the
sequence of intended changes. There is nothing to rubber-stamp yet, only a
commitment to a path.

The second is equivalence sign-off, after the suite is green. Reading "58 of 58
passed" and deciding it is sufficient evidence is itself the judgment, not a
formality the exit code settles. For a cutover this entangled, signing off meant a
person taking responsibility for the claim that a real-socket run against both
backends, in both flag states, with a negative check, was the right coverage for
the stakes. The agent drafts; the gate checks; a person owns the two decisions
that carry the risk.

## What this buys modernization

Under a traditional lifecycle, an extraction is measured in quarters per service,
because it has to pass a row of serially scheduled human handoffs to be trusted.
The ADLC keeps the two decisions that hold the risk and collapses the
rest into a loop an agent runs in an afternoon, because the evidence those
dropped handoffs used to produce by hand is now a suite that re-runs on every
candidate build. The strangler-fig shape keeps each step reversible; the
equivalence gate proves each step matches the behavior it replaced. Six seams
later, the proxy routes
nothing to the monolith, the flags are gone, and the only irreversible step,
deleting the monolith's code, happens once, at the end, after every cutover it
depended on was already proven.

## Source project

This post is mined from the
[Modernizing Enterprise Applications (Python)]({{ page.canonical_project.url }})
build, specifically
[The strangler-fig pattern](https://patterncatalyst.github.io/modernizing-enterprise-applications-python/docs/14-strangler-fig-pattern/),
[Extraction 1: the review service](https://patterncatalyst.github.io/modernizing-enterprise-applications-python/docs/15-extraction-1-review-service/),
and [The ADLC safety net](https://patterncatalyst.github.io/modernizing-enterprise-applications-python/docs/07-adlc-safety-net/),
which carries the full chunked-body incident and the two-gate argument.

The code shown is the
[strangler proxy](https://github.com/patterncatalyst/modernizing-enterprise-applications-python/blob/main/examples/01-strangler-proxy/app/main.py)
([routing tests](https://github.com/patterncatalyst/modernizing-enterprise-applications-python/blob/main/examples/01-strangler-proxy/tests/test_routing.py)),
the
[extracted review service](https://github.com/patterncatalyst/modernizing-enterprise-applications-python/blob/main/examples/02-review-service/app/routes.py),
and the
[frozen Django monolith](https://github.com/patterncatalyst/modernizing-enterprise-applications-python/blob/main/examples/00-monolith/review/views.py)
it replaced. The `reference/monolith-before` branch and the `stage/01-review`
through `stage/06-order` snapshots let you read each seam side by side.

The conceptual companion to this post,
[why an ADLC replaces the SDLC's slowness and not its safety]({{ site.baseurl }}/2026/10/18/adlc-replaces-slowness-not-safety/),
is drawn from the
[Quarkus edition](https://patterncatalyst.github.io/modernizing-enterprise-applications/)
of the same build.
Full projects:
[Python](https://patterncatalyst.github.io/modernizing-enterprise-applications-python/)
· [Quarkus](https://patterncatalyst.github.io/modernizing-enterprise-applications/).
More field notes on the [Pattern Catalyst hub]({{ site.hub_url }}).
