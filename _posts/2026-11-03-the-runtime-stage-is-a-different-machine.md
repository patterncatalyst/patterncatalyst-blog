---
title: "The Runtime Stage Is a Different Machine"
date: 2026-11-03
author: "Pattern Catalyst"
tags: [containers, multi-stage-build, security, kubernetes]
categories: [cloud-native]
canonical_project:
  name: "Cloud-Native Design Patterns"
  repo: "patterncatalyst/cloud-native-design-patterns"
  url: "https://patterncatalyst.github.io/cloud-native-design-patterns/"
excerpt: "A single-stage Containerfile ships the compiler to production. Multi-stage builds treat the build box and the run box as two different machines, and that one separation is what later lets you pick a hardened runtime base, run as non-root, and ship something a cluster can pull fast."
---

A single-stage Containerfile makes one quiet assumption: that the machine which
compiled your service is a fine machine to run it on. It is the same image, so
it carries the same contents. The compiler that produced the binary rides along
to production. So does the dependency resolver, its download cache, the source
tree, and every build intermediate. None of it executes once the service is up.
All of it gets pulled from the registry, scanned for CVEs, and stored on every
node that schedules the pod.

The build box and the run box are not the same machine. They have different
jobs, different contents, and different threat models. A multi-stage build is
how you say that in a Containerfile: a `build` stage that may hold anything it
needs to produce the artifact, and a `runtime` stage that holds the artifact and
the smallest base that can execute it. One `COPY --from` crosses the boundary.
Everything else in the build stage is discarded.

{% include excalidraw.html
   file="two-stage-boundary"
   alt="Two framed regions side by side. The left region, labelled build stage (discarded), stacks four boxes: compiler plus toolchain, dependency cache, source tree plus intermediates, and at the bottom a highlighted built artifact (jar, binary, or venv). The right region, labelled runtime stage (shipped), holds a hardened runtime base (minimal, non-root) and the same highlighted artifact with nothing else. A single amber COPY --from arrow carries the artifact from the build stage into the runtime stage. A caption under the left reads everything here is thrown away; under the right, this is what the registry stores."
   caption="One image, two machines: the build stage may hold the whole toolchain, but only the artifact crosses into the image you ship" %}

## The same shape in every runtime

The useful thing about this split is that it is not language-specific. The
[Cloud-Native Design Patterns]({{ page.canonical_project.url }}) examples build
the same service in Quarkus, Go, Python, and .NET, and every Containerfile
collapses to the identical two-stage shape: build with whatever the language
needs, then copy one artifact onto a minimal runtime base.

Java, where the build stage needs a full JDK and Maven and the runtime needs
only a JRE:

```dockerfile
# cloud-native-design-patterns/examples/01-cloud-native-principles/quarkus/Containerfile
FROM registry.access.redhat.com/ubi10/openjdk-25:1.24-15 AS build
USER root
RUN microdnf install -y maven && microdnf clean all
USER default
WORKDIR /home/default
COPY pom.xml .
RUN --mount=type=cache,target=/home/default/.m2,uid=185 mvn dependency:go-offline -B
COPY src src
RUN --mount=type=cache,target=/home/default/.m2,uid=185 mvn package -DskipTests -B

FROM registry.access.redhat.com/ubi10/openjdk-25-runtime:1.24-15
WORKDIR /deployments
COPY --from=build --chown=185:0 --chmod=g=u /home/default/target/quarkus-app/ ./
USER 185
EXPOSE 8080
CMD ["java", "-jar", "quarkus-run.jar"]
```

Go, where the build stage needs the compiler and the runtime needs nothing
more, since a static binary carries its own dependencies:

```dockerfile
# cloud-native-design-patterns/examples/01-cloud-native-principles/go/Containerfile
FROM docker.io/library/golang:1.27.2 AS builder
WORKDIR /src
COPY go.mod go.sum* ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags='-s -w' -o /app ./cmd/server

FROM registry.access.redhat.com/ubi10/ubi-minimal:10.2-1791444377 AS runtime
RUN microdnf install -y curl-minimal && microdnf clean all
COPY --from=builder --chown=1001:0 --chmod=g=u /app /usr/local/bin/app
USER 1001
EXPOSE 8080
ENTRYPOINT ["/usr/local/bin/app"]
```

Python, where the build stage resolves a locked dependency set into a virtual
environment and the runtime copies that venv without ever installing a package
manager:

```dockerfile
# cloud-native-design-patterns/examples/01-cloud-native-principles/python/Containerfile
FROM registry.access.redhat.com/ubi10/python-314-minimal:10.2-1791464217 AS build
USER 0
RUN pip install --no-cache-dir uv==0.13.0
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/uv-cache uv sync --locked --no-dev

FROM registry.access.redhat.com/ubi10/python-314-minimal:10.2-1791464217
COPY --from=build --chown=1001:0 --chmod=g=u /opt/app-root/src/.venv /opt/app-root/src/.venv
COPY --chown=1001:0 --chmod=g=u main.py .
ENV PATH="/opt/app-root/src/.venv/bin:$PATH"
USER 1001
CMD ["opentelemetry-instrument", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

Three languages and three toolchains collapse to one pattern. The second `FROM`
opens a fresh filesystem, and `COPY --from` is the only channel back to the build
stage. What rides across in each is small: a `quarkus-app/` directory, a single
static binary, a resolved venv. What stays behind is everything else: Maven, the
Go compiler, `uv`, every cache and header and object file. The build stage can be
as heavy as it wants, because it is not the thing you ship.

## The split is what lets you harden the runtime

Shrinking the image is the obvious payoff, and it is real. The deeper one is
that the separation is a *precondition* for a hardened runtime base. It is what
makes one possible at all. You cannot put a distroless base under a single-stage build,
because a single-stage build still has to run the compiler, and a distroless
image has no shell and no package manager to run it with.

The [Hummingbird tutorial](https://patterncatalyst.github.io/hummingbird-tutorial/docs/04-multi-stage-builds/)
makes this concrete with Red Hat's near-zero-CVE images. Its runtime images are
distroless: no `/bin/sh`, no package manager, nothing to interpret a `RUN` line.
Try to `RUN` anything in the runtime stage and Buildah fails outright with
`executable file '/bin/sh' not found`. That constraint is only livable because
the build stage already did all the running:

```dockerfile
# hummingbird-tutorial/examples/go-example/Containerfile
# Stage 1: compile with the Hummingbird Go builder
FROM quay.io/hummingbird/go:1.26-builder AS builder
USER 1001
WORKDIR /build
ENV HOME=/build GOCACHE=/build/.cache/go-build
COPY --chown=1001:1001 go.mod main.go ./
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o app .

# Stage 2: the distroless Hummingbird Go runtime
# glibc, ca-certificates, a non-root UID 1001, and not much else.
# No Go toolchain. That is what the builder was for.
FROM quay.io/hummingbird/go:1.26
WORKDIR /app
COPY --from=builder --chown=1001:1001 /build/app ./app
USER 1001
EXPOSE 8080
CMD ["./app"]
```

The payoff is a different number than image size. **The CVE surface that ships
to production is the runtime image alone; the builder never travels.** It can sit on a
full UBI base with Maven and a compiler and whatever that drags in, and none of
its vulnerabilities reach the deployed artifact, because the builder never leaves
the build host. The Hummingbird Quarkus example leans on exactly this: it builds
on a full UBI OpenJDK image for the convenience of a pre-installed Maven, then
ships on a distroless JRE, and notes in its own comments that the UBI builder's
"larger CVE surface doesn't reach production." A static Go binary on the
distroless Go runtime lands around 30 MB, with an attack surface that has no
shell for an attacker to drop into.

{% include excalidraw.html
   file="split-pays-off-three-ways"
   alt="A highlighted box on the left labelled build / runtime split fans out with three amber arrows to three result boxes: smaller image (faster cold-start pulls), hardened runtime base (smaller CVE surface), and non-root artifact (restricted SCC ready). All three result boxes then feed with arrows into a box on the right labelled Kubernetes cluster. A caption reads: the same split that shrinks the image hardens it."
   caption="One decision, three payoffs: the separation that shrinks the image is also what lets you harden the base and run unprivileged" %}

## What it buys at deploy time

A cluster is where these payoffs get collected. The
[minikube data-mesh final project](https://patterncatalyst.github.io/minikube-on-fedora/docs/17-capstone/)
deploys a fleet of services, and each one uses the same two-stage build for a
reason that shows up under orchestration. Here is its order service, with the
boundary annotated in its own comments:

```dockerfile
# minikube-on-fedora/examples/17-capstone/services/order-service/Containerfile
# Builder (runs as root; image discarded)
FROM registry.access.redhat.com/ubi10/python-314-minimal:10.2-1791464217 AS builder
USER 0
WORKDIR /build
RUN pip install --no-cache-dir "poetry==1.8.4" "poetry-plugin-export==1.8.0"
COPY pyproject.toml poetry.lock* ./
RUN poetry export --without-hashes --only main -f requirements.txt -o requirements.txt \
    && python -m venv /opt/venv \
    && /opt/venv/bin/pip install --no-cache-dir -r requirements.txt

# Runtime (non-root)
FROM registry.access.redhat.com/ubi10/python-314-minimal:10.2-1791464217 AS runtime
WORKDIR /opt/app-root/src
COPY --from=builder --chown=1001:0 /opt/venv /opt/venv
COPY --chown=1001:0 app/ ./app/
ENV PATH="/opt/venv/bin:$PATH"
USER 1001:0
EXPOSE 8080
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

The split does three jobs here that a single stage cannot. The builder runs as
root so it can write the venv under `/opt`, which the comment calls out as having
"no security cost" precisely because that stage is thrown away; only the runtime
stage enforces a non-root user. That non-root user, `1001:0`, is what lets the
pod schedule under a restricted security context on Kubernetes or OpenShift
without an exception. The smaller image pulls faster, which matters most on the
cold scale-up KEDA triggers when a queue backs up, since a node that doesn't
already have the image has to fetch it before the pod can start. And Poetry,
the export plugin, and the pip cache that resolved the dependencies never reach
the node at all.

None of that is a separate feature you bolt on. It is the same build/runtime
separation, read from the cluster's side of the fence.

## The rules that make it hold

Across all three projects the discipline is the same short list:

- **Put the runtime base in the last `FROM`.** The final stage is what ships, so
  it gets the minimal or distroless image. Everything heavier lives in a stage
  that ends before it.
- **`COPY` into the runtime stage; never `RUN`.** If a step needs to execute
  something (`pip install`, `go build`, `mvn package`), it belongs in the
  builder. A distroless runtime has no shell to run it anyway, which turns this
  rule into a compile error instead of a code review note.
- **`--chown` the artifact to a non-root user.** Copy to `1001:0` (or the base's
  default non-root UID) so the image is restricted-SCC ready without a runtime
  `chown`.
- **Cache dependencies in the builder so they stay out of the image.**
  `--mount=type=cache` and a dependency-manifest-first `COPY` keep rebuilds fast
  without leaving a package cache in the shipped layers.
- **Pin the base tags.** A specific tag on both stages keeps a silent
  major-version bump out of your build.

A single-stage build breaks none of these rules. It just ships a machine
you did not mean to ship. Treat the runtime stage as a different machine from the
start, and the small image, the smaller CVE surface, and the unprivileged pod all
fall out of the one decision.

## Source project

This post is mined from the container Containerfiles across the
[Cloud-Native Design Patterns]({{ page.canonical_project.url }}) examples, whose
[Cloud-Native Principles chapter](https://patterncatalyst.github.io/cloud-native-design-patterns/docs/01-cloud-native-principles/)
frames leaning on the platform and shipping the smallest runnable artifact. The
per-runtime Containerfiles shown are
[`quarkus/Containerfile`](https://github.com/patterncatalyst/cloud-native-design-patterns/blob/main/examples/01-cloud-native-principles/quarkus/Containerfile),
[`go/Containerfile`](https://github.com/patterncatalyst/cloud-native-design-patterns/blob/main/examples/01-cloud-native-principles/go/Containerfile), and
[`python/Containerfile`](https://github.com/patterncatalyst/cloud-native-design-patterns/blob/main/examples/01-cloud-native-principles/python/Containerfile).

The distroless-runtime argument and the Go/Quarkus examples are from
**Multi-stage builds** in the
[Hummingbird Tutorial](https://patterncatalyst.github.io/hummingbird-tutorial/docs/04-multi-stage-builds/)
([`go-example/Containerfile`](https://github.com/patterncatalyst/hummingbird-tutorial/blob/main/examples/go-example/Containerfile),
[`quarkus-example/Containerfile`](https://github.com/patterncatalyst/hummingbird-tutorial/blob/main/examples/quarkus-example/Containerfile)).
The deploy-time service is the order service from the
[minikube data-mesh final project](https://patterncatalyst.github.io/minikube-on-fedora/docs/17-capstone/)
([`order-service/Containerfile`](https://github.com/patterncatalyst/minikube-on-fedora/blob/main/examples/17-capstone/services/order-service/Containerfile)).

Full projects:
[Cloud-Native Design Patterns](https://patterncatalyst.github.io/cloud-native-design-patterns/)
· [Hummingbird Tutorial](https://patterncatalyst.github.io/hummingbird-tutorial/)
· [Minikube on Fedora](https://patterncatalyst.github.io/minikube-on-fedora/).
More field notes on the [Pattern Catalyst hub]({{ site.hub_url }}).
