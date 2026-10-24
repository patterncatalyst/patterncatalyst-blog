---
title: "A Mean Is Not a Latency, But Little's Law Needs One Anyway"
date: 2026-10-23
author: "Pattern Catalyst"
tags: [latency, performance, littles-law, amdahl, modelling, cpp, java, python]
categories: [performance]
canonical_project:
  name: "Low-Latency Engineering in C++, Python & Java"
  repo: "patterncatalyst/low-latency-engineering"
  url: "https://patterncatalyst.github.io/low-latency-engineering/"
excerpt: "A mean is the wrong number to report as a latency, and it is also the one number Little's Law and Amdahl's Law cannot run without. The fix is to quarantine it: latencies stay percentiles, means live in a labelled model block and nowhere else."
---

Report a mean latency and you have told your reader almost nothing true. The
average sits above the typical request and far below the tail, so it describes
neither the request most users get nor the one that breaks the SLO. Every
latency chapter worth reading says the same thing: publish percentiles, never a
mean.

And yet two of the most useful results in performance work, Little's Law and
Amdahl's Law, take a mean as their input and refuse to run without one. That is
the tension this post resolves. A mean is the wrong number to *report* and the
right number to *model with*. The discipline that keeps both facts true at once
is to give the mean exactly one place to live: a labelled model block, separate
from the latency report, and nowhere else.

This is mined from Chapter 2 of our low-latency engineering book, where the rule
is enforced in code across C++, Java, and Python. The running system is a parcel
sortation line: a conveyor carries a parcel from scanner to diverter, and the
spine metric is the scan-to-divert latency, sense to decide to act.

## A mean is a bad latency

Start with why the average is the wrong thing to report. Here is the residence
time `W` measured on the C++ consumer in Chapter 2, the time from a parcel's
intended emit to the handler marking it done:

{% include excalidraw.html file="mean-neither-typical-nor-tail"
   alt="A horizontal spectrum of W, residence time, on the C++ consumer, with four markers left to right: p50 at 0.78 ms labelled the typical parcel; W-bar the mean at 1.05 ms labelled the model input and drawn in the amber accent; p99 at 4.35 ms; and p99.9 at 8.5 ms labelled the tail. An arrow beneath points toward slower. A slate emphasis box reads that the mean is useless as a latency because it is neither the typical parcel nor the tail, yet essential as L equals lambda times W-bar, the only value that ties queue length to rate."
   caption="Figure 1: The mean residence time, 1.05 ms, sits above the typical parcel (p50, 0.78 ms) and far below the tail (p99, 4.35 ms). As a latency it describes no real parcel. It has exactly one job, and Figure 2 shows it." %}

The mean is 1.05 ms. The p50 is 0.78 ms and the p99 is 4.35 ms. Report the mean
and a reader concludes parcels take about a millisecond, which overstates the
common case by a third and hides the tail by a factor of four. The mean is not a
summary of the distribution. It is a single arithmetic fact about it, and a
misleading one to hand someone who wants to know how slow things get.

So the book bans it from every latency report. Reports carry p50, p99, and
p99.9, never an average. The rule is absolute enough that it is written into the
project's house constraints: a time-average may appear only as a labelled
modelling input, never as a reported latency.

## A mean is the one thing the laws need

Now the other side. Little's Law states that for a stable system the number of
items inside it equals the arrival rate times the time each item spends inside:

```
L = lambda x W
```

Point it at the belt and it predicts the work in flight before you run anything.
The conveyor carries a parcel from scanner to diverter in 480 ms, and at 1,000
parcels per second that is `1000 x 0.48 = 480` parcels on the belt at once, from
two parameters and no measurement. Point it at the handler instead and `W` is
the mean residence time. There is no percentile form of this law. `W` is a
time-average by definition, because only the average ties the long-run queue
length to the arrival rate. The p50 will not do it; the p99 will not do it. The
law needs the mean or it needs nothing.

The helper is a single line in each language, and its name carries the warning:

```cpp
// examples/02-littles-amdahl/cpp/include/lowlat/ch02/model.hpp
[[nodiscard]] constexpr double littles_L(double lambda_per_s, double w_seconds) noexcept {
  return lambda_per_s * w_seconds;
}

// Amdahl's Law: with serial fraction s, the speedup on n workers.
[[nodiscard]] constexpr double amdahl_speedup(double serial_fraction, unsigned workers) noexcept {
  return 1.0 / (serial_fraction + (1.0 - serial_fraction) / static_cast<double>(workers));
}
```

Amdahl's Law has the same shape of dependency. If a fraction `s` of a one-worker
run cannot be parallelised, then `N` workers give a speedup of
`1 / (s + (1 - s) / N)`, and no number of workers passes `1/s`. That `s` is also
a time fraction, another average: the share of wall time spent in the serial
section, averaged over the run. Chapter 2 measures it rather than guessing,
timing every lock section across five calibration runs and dividing the summed
lock time by the summed wall time:

```cpp
// examples/02-littles-amdahl/cpp/src/amdahl.cpp
// Serial fraction s = time inside the lock / wall time, at N = 1. Ratio of sums
// over the calibration runs.
std::int64_t lock_total = 0, wall_total = 0;
for (int i = 0; i < o.calibration_runs; ++i) {
  const Run r = run_batch<true>(plan, o, 1);
  lock_total += r.in_lock_ns;
  wall_total += r.wall_ns;
}
const double s = static_cast<double>(lock_total) / static_cast<double>(wall_total);
```

Both laws eat a time-average and predict structure. Neither one has a percentile
anywhere in it. If you have forbidden means from the codebase, you have also
forbidden the two cheapest predictions in performance engineering.

## Quarantine, don't ban

The resolution is not to relax the ban. It is to give the mean a room of its own.
After each run, the consumer writes its percentile report as usual, and beside it
writes a separate `model` block that is the only structure in the program allowed
to hold an average. Every field name states what it is:

```cpp
// examples/02-littles-amdahl/cpp/src/littles.cpp
// --- the model block: the only place a mean appears ---------------------
const double lambda   = static_cast<double>(o.rate_hz);
const double w_mean_s = static_cast<double>(handler.w_sum_ns() / received) / 1e9;
const double l_sampled   = static_cast<double>(sampled.sum_in_system) / sampled.samples;
const double l_predicted = lowlat::ch02::littles_L(lambda, w_mean_s);
const double rel_err     = (l_sampled - l_predicted) / l_predicted;
const nlohmann::json model = {
    {"law", "Little: L = lambda * W"},
    {"lambda_per_s", lambda},
    {"L_sampled_time_average", l_sampled},
    {"W_mean_seconds_modelling_input", w_mean_s},
    {"predicted_L", l_predicted},
    {"relative_error", rel_err},
};
```

Read the field names. It is not `w_mean_ms`, it is
`W_mean_seconds_modelling_input`. The label is part of the value. A later reader
grepping the reports for a latency to quote cannot pick this one up by accident,
because the name tells them it is a model input, not a result. The latency report
and the model block are written next to each other and are never confused,
because the data structure itself refuses to confuse them.

{% include excalidraw.html file="mean-model-block"
   alt="A diagram of one latency report for the consumer with 10,000 parcels at 1000 per second. Inside a report frame sit two boxes. The left box, reported latency, holds percentiles only: p50 0.78 ms, p99 4.35 ms, p99.9 8.5 ms, and the note no mean here. The right box, drawn in the amber accent and labelled the only mean, is the model block holding W-bar 1.05 ms and lambda 1000 per second, a labelled modelling input. An amber arrow carries W-bar out of the model block into a slate box for Little's Law, L equals lambda times W-bar, which feeds a result box reading predicted L 1.05, sampled L 1.05, error plus 0.5 percent. A caption across the top reads that a mean is never a latency and appears once, labelled, as a model input."
   caption="Figure 2: One run produces two things. The latency report carries percentiles and no mean. The model block is the only structure that holds an average, and it feeds Little's Law, not the SLO." %}

The same optional block flows through all three languages. The measurement
library's report writer takes an optional model argument and writes it beside the
percentiles, so C++ hands it an `nlohmann::json`, Python a dict, and Java a map,
and the key is omitted when a run has no model to attach. The quarantine
is a property of the shared tooling, not a convention each program is trusted to
follow.

## The model predicts, the measurement checks

Keeping the mean in its own block buys more than tidiness. It lets the run check
the model against reality in the same pass. The consumer samples the live queue
length `L` from a separate thread while the per-parcel clock records `W`, so `L`
and `W` arrive by independent mechanisms. If `L = lambda x W-bar` holds, both are
confirmed at once. In Chapter 2 the sampled `L` matched the prediction within 0.3
to 0.5 percent in C++ and within 1.3 to 4.3 percent in Java.

The sharpest demonstration that the discipline works comes from where the
model fails. Amdahl's Law knows about the lock, because the lock is what `s`
measures. It knows nothing about Python's Global Interpreter Lock, because the
GIL is not in the formula. So run the phased workload on Python threads and watch
the prediction break:

| Workers (N=4) | Predicted | Measured |
|---|---|---|
| C++ | ~3.1x | 2.83 to 2.85x |
| Java | ~3.1x | 2.74 to 2.75x |
| Python processes | ~3.0x | 2.92 to 2.97x |
| Python threads | ~3.1x | **0.47 to 0.50x** |

CPU-bound Python threads under the GIL do not merely fail to speed up. They run
more than twice as slow at four workers as at one, because every barrier and lock
hand-off also pays a GIL hand-off, and the interpreter still runs one thread at a
time. Processes, one interpreter each, track the prediction closely. The threads
row is a negative control: the model misses it wildly, and that miss is the
signal. A clean, labelled model is one you can hold a measurement against and
learn exactly which mechanism you left out. A mean smuggled into a latency report
could never tell you that, because you would never have known which number was a
claim and which was an input.

## The takeaway

The rule is small and it survives contact with real code:

- A mean is a poor summary of a latency distribution. It is neither the typical
  request nor the tail. Report percentiles.
- A mean is the required input to Little's Law and Amdahl's Law. There is no
  percentile substitute; the laws tie averages to structure.
- Resolve the tension by quarantine, not prohibition. Latencies stay
  percentiles in the report; means live in a labelled model block whose field
  names mark them as inputs.
- A clean model pays off when it is wrong. The Python-threads miss is only
  legible because the mean was never allowed to pose as a result.

## Source project

This post is mined from **Chapter 2, "Modelling with Little's Law and Amdahl's
Law"**, of
[Low-Latency Engineering in C++, Python & Java]({{ page.canonical_project.url }}),
a chaptered, measured book that teaches latency engineering through runnable
C++20/23, Python 3.14, and Java 25 examples on a parcel sortation line. Read the
chapter in full at
[the chapter page](https://patterncatalyst.github.io/low-latency-engineering/docs/02-littles-amdahl/),
including the sampler that measures `L`, the barrier-phased Amdahl workload, and
the free-running-lock variant that beats the Amdahl curve.

Full files for the snippets above, at the `r02.x` release:
[`model.hpp`](https://github.com/patterncatalyst/low-latency-engineering/blob/r02.x/examples/02-littles-amdahl/cpp/include/lowlat/ch02/model.hpp),
[`littles.cpp`](https://github.com/patterncatalyst/low-latency-engineering/blob/r02.x/examples/02-littles-amdahl/cpp/src/littles.cpp),
and
[`amdahl.cpp`](https://github.com/patterncatalyst/low-latency-engineering/blob/r02.x/examples/02-littles-amdahl/cpp/src/amdahl.cpp).
The [repository]({{ page.canonical_project.repo | prepend: "https://github.com/" }})
has the Java and Python counterparts and the `verify.sh` that asserts every
number above.

More posts mined from the catalog are indexed on the
[Pattern Catalyst hub]({{ site.hub_url }}).

*Numbers quoted here were measured on a bare-metal 8-core/16-thread Intel laptop
CPU, Fedora 44, governor `powersave`, untuned, as recorded in the chapter. A
measurement is only comparable against its own machine profile.*
