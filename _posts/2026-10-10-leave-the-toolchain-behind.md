---
title: "Leave the Toolchain Behind: A C++ Service in 26 MB Instead of 689"
date: 2026-10-10
author: "Pattern Catalyst"
tags: [cpp, containers, multi-stage-build, ubi, image-size]
categories: [cloud-native]
canonical_project:
  name: "Optimizing Modern C++ with Containers"
  repo: "patterncatalyst/cpp-container-optimization-tutorial"
  url: "https://patterncatalyst.github.io/cpp-container-optimization-tutorial/"
excerpt: "A 4 MB C++ binary routinely ships inside a 689 MB image. The fix isn't to shrink the toolchain, it's to leave it behind. Multi-stage builds plus the right UBI runtime tier get the same service down to 26 MB, and the tier you pick is really an incident-response decision."
---

A C++ service compiles to a 4 MB binary and ships inside a 689 MB image. The
binary is 0.6% of what gets pulled from the registry, scanned for CVEs, and
walked on cold start. The other 685 MB is the compiler, the package-manager
cache, a pile of object files, and the source tree, none of which the running
program ever touches.

The instinct is to go hunting for a smaller base image or a leaner compiler
install. That is the wrong lever. The toolchain is not bloat to be trimmed; it
is build-time machinery that has no business being in the runtime image at all.
The fix is to leave it behind.

## What the 685 MB is doing

A first-pass Containerfile builds and runs in the same image. Install the
compiler, copy the source, compile, set an entrypoint. It works, and it
produces this:

| What | How much | Why it's in there |
|---|---|---|
| `gcc` / `gcc-c++`, headers, linker | ~400 MB | needed to compile, not to run |
| Conan cache (sources, variants, binaries) | ~200 MB | needed to resolve deps, not to use them |
| CMake configure cache, `.o` files, `.a` archives | ~80 MB | build intermediates |
| Source tree | ~5 MB | needed to be compiled, not executed |

Every one of those lines is present only because the build happened in the image
you shipped. None of it is a dependency of the program. The compiler does not
run in production. The Conan cache resolved your dependencies months ago. The
`.o` files were consumed by the linker and never read again.

{% include excalidraw.html
   file="leave-the-toolchain-behind"
   alt="A comparison of two build strategies. On the left, a single-stage image stacks the compiler and Conan cache (about 600 MB), object files and source tree (about 85 MB), and the 4 MB binary, and the entire 689 MB ships. On the right, a build stage holds the discarded toolchain and produces the 4 MB binary, a COPY --from=build arrow carries only the binary into a runtime image built on a 14 MB ubi-micro base, for 26.4 MB total."
   caption="The toolchain builds the binary; only the binary needs to travel to production" %}

## Multi-stage builds: the whole trick is one line

A multi-stage Containerfile declares more than one `FROM`. Each `FROM` begins a
fresh image. You name the build image with `AS`, compile in it, then start a
second image for runtime and copy just the finished binary across. Only the
final stage is tagged; the build stage is discarded the moment the build
finishes.

Here is the real build stage from the tutorial's demo, lightly trimmed:

```dockerfile
# Containerfile.ubi-multistage — build stage
ARG UBI_VERSION=10.2
FROM registry.access.redhat.com/ubi10/ubi:${UBI_VERSION} AS build
RUN dnf install -y --setopt=install_weak_deps=False \
        gcc gcc-c++ cmake ninja-build git \
    && dnf clean all
WORKDIR /src
COPY src/ ./src/
COPY CMakeLists.txt CMakePresets.json ./
RUN cmake --preset release \
 && cmake --build --preset release -j"$(nproc)" \
 && strip --strip-all build/release/demo-svc
```

Then the runtime stage starts over from a smaller base and copies one file:

```dockerfile
# Containerfile.ubi-multistage — runtime stage
FROM registry.access.redhat.com/ubi10/ubi-minimal:${UBI_VERSION} AS runtime
RUN microdnf install -y --setopt=install_weak_deps=0 libstdc++ \
 && microdnf clean all
WORKDIR /app
COPY --from=build /src/build/release/demo-svc /app/demo-svc
USER 1001:1001
EXPOSE 8080
ENTRYPOINT ["/app/demo-svc"]
```

`COPY --from=build /src/build/release/demo-svc /app/demo-svc` is the entire
mechanism. It reaches into the finished `build` image and lifts out one file.
The compiler, the Conan cache, the object files, and the source tree stay behind
in a stage that is never tagged and never pushed. They die with stage one.

Build *time* barely changes, because you are still compiling the same code. What
changes is what ships. On `ubi-minimal` this gets the demo from 689 MB to about
114 MB. The next decision takes it the rest of the way.

## Picking the runtime base is an incident-response decision

Red Hat's Universal Base Image ships in three runtime tiers, and the gap between
them is not about megabytes:

| Tier | Size | Shell | Package manager | Pick it for |
|---|---|---|---|---|
| `ubi10/ubi` | ~210 MB | `bash` | `dnf` | build stages, debug images |
| `ubi10/ubi-minimal` | ~100 MB | `bash` | `microdnf` | most production C++ services |
| `ubi10/ubi-micro` | ~14 MB | none | none | security-sensitive, measured deployments |

It is tempting to read that table top to bottom and reach for the smallest row.
Resist that. The real question is what your on-call playbook does when the
service misbehaves at 3 a.m.

`ubi-minimal` keeps a working `bash` and `microdnf`, so `podman exec -it ...
bash` drops you into a live container and you can install `ldd` or poke around.
That convenience costs roughly 85 MB over `ubi-micro`.

`ubi-micro` is about 14 MB before your binary lands. There is no shell, no
`dnf`, no `ldd`, no `strace`. You cannot `exec` a `bash` into it because there
is no `bash` to exec. If your binary needs `libssl` or `libstdc++`, you either
link them statically or `COPY` them explicitly from the build stage. The
tutorial's `ubi-micro` variant goes fully static and lands at **26.4 MB**:

```dockerfile
# Containerfile.ubi-micro — runtime stage (binary is statically linked)
FROM registry.access.redhat.com/ubi10/ubi-micro:${UBI_VERSION} AS runtime
WORKDIR /app
COPY --from=build /src/build/release-static/demo-svc /app/demo-svc
# ubi-micro has no /etc/passwd lookups to worry about
USER 1001:1001
EXPOSE 8080
ENTRYPOINT ["/app/demo-svc"]
```

So the decision is: if your playbook starts with "shell into the container,"
ship `ubi-minimal`. If your playbook starts with "attach an ephemeral debug
sidecar that shares the PID namespace and carries gdb," ship `ubi-micro`. The
sidecar posture is where most production deployments land, and it is what makes
the 26 MB image safe to operate rather than merely small.

## A 26 MB image is opaque, so write down what's inside it

Shrinking the image removes your ability to interrogate it. On `ubi-micro` you
cannot run `dnf list installed`, you cannot `ldd` the binary, you cannot ask the
image what it was built against. Six months from now a CVE drops against a range
of `libstdc++` versions and nobody can tell from the image whether you are
affected.

The answer is to write the toolchain identity into the image metadata at build
time, where a small image can still carry it cheaply:

```dockerfile
LABEL org.opencontainers.image.title="demo-svc" \
      org.opencontainers.image.revision="${GIT_SHA}" \
      tutorial.toolchain="gcc-14" \
      tutorial.linkage="static" \
      tutorial.lto="thin"
```

`podman inspect <image> | jq '.[0].Config.Labels'` reads them back at incident
time without running anything. The standard `org.opencontainers.image.*` keys
cover provenance; a project-specific prefix records the things that bite C++
specifically: which libc and libstdc++ the binary expects, which
micro-architecture it was tuned for, whether LTO or PGO is in play. The image is
too small to hold the toolchain, so it holds the toolchain's name tag instead.

## The trap that makes this a C++ problem

A Go binary carries its runtime inside itself; an Uberjar carries the JVM's
class libraries. C++ does not work that way. A typical C++ binary depends on a
specific `libstdc++.so.6` and a specific `libc.so.6`, and those dependencies are
implicit right up until they are fatal.

The fatal version looks like this. Build the service on one distro's glibc and
run it on an older one, and the dynamic linker goes looking at startup for a
symbol version the runtime's `libc.so.6` does not carry:

```
/app/demo-svc: /lib64/libc.so.6: version `GLIBC_2.41' not found (required by /app/demo-svc)
```

The tutorial reproduces this on demand by building on Fedora 42 (glibc 2.41) and
running on `ubi-micro` (glibc 2.39). The subtle part: compiling with
`-static-libstdc++` does *not* save you, because that statically links the C++
runtime while glibc stays dynamic, so a newer glibc symbol baked in at build
time is still resolved against whatever `libc.so.6` the runtime ships. Two fixes
work. Link glibc statically too (`-static`), which is what the 26.4 MB
`ubi-micro` variant does and why it has no libc to mismatch. Or, more directly,
use the same UBI release for both stages so the build glibc and the runtime
glibc are identical by construction. UBI-10-on-UBI-10 sidesteps the whole class
of failure without any static linking at all.

That is the same shape as the micro-architecture pitfall, where a binary built
with `-march=native` on an AVX-512 host crashes with `SIGILL` on an older
runtime CPU. In both cases the build environment and the runtime environment
disagree, and the image strategy is what makes that disagreement visible or
invisible.

## The takeaway

Image size for a C++ service is not a packaging afterthought. It is a decision
about which dynamic libraries travel with the binary, which ones you assume at
the runtime end, and how you will diagnose the thing once it is small enough to
be opaque. Multi-stage builds leave the toolchain behind. The UBI tier you pick
encodes your incident-response posture. Labels buy back the introspection the
small image gave up. Get those three right and the same 4 MB binary that was
shipping in 689 MB ships in 26.

## Source project

This post is mined from **§4, "Container Strategy: UBI, ubi-micro,
multi-stage"** of
[Optimizing Modern C++ with Containers](https://patterncatalyst.github.io/cpp-container-optimization-tutorial/docs/04-image-strategy/),
which walks through the full argument, the layer-caching ordering rules, and the
production diagnostic recipe for reading a small image.

The code samples are drawn from the tutorial's runnable
[`demo-01-image-strategy`](https://github.com/patterncatalyst/cpp-container-optimization-tutorial/tree/main/examples/demo-01-image-strategy),
which builds the same service four ways and prints the verified sizes:
[`Containerfile.single-stage-naive`](https://github.com/patterncatalyst/cpp-container-optimization-tutorial/blob/main/examples/demo-01-image-strategy/Containerfile.single-stage-naive) (689 MB),
[`Containerfile.ubi-multistage`](https://github.com/patterncatalyst/cpp-container-optimization-tutorial/blob/main/examples/demo-01-image-strategy/Containerfile.ubi-multistage) (114 MB),
[`Containerfile.ubi-micro`](https://github.com/patterncatalyst/cpp-container-optimization-tutorial/blob/main/examples/demo-01-image-strategy/Containerfile.ubi-micro) (26.4 MB), and
[`Containerfile.ubi-micro-glibc-mismatch`](https://github.com/patterncatalyst/cpp-container-optimization-tutorial/blob/main/examples/demo-01-image-strategy/Containerfile.ubi-micro-glibc-mismatch) (the one engineered to fail).

Full project: [site](https://patterncatalyst.github.io/cpp-container-optimization-tutorial/)
· [repo](https://github.com/patterncatalyst/cpp-container-optimization-tutorial).
