---
title: Native Histograms
sort_rank: 5
---

// Final completion check by going through:
// - The design docs, especially appendix to original design doc.
// - Grafana/Mimir docs AKA Krajo's docs.
//   - https://grafana.com/docs/mimir/latest/send/native-histograms/
//   - https://grafana.com/docs/mimir/latest/visualize/native-histograms/
//   - Source: https://github.com/grafana/mimir/tree/main/docs/sources/mimir/send/native-histograms has the LaTeX math
// - open issues
// - Carrie's docs:  https://docs.google.com/document/d/1VhtB_cGnuO2q_zqEMgtoaLDvJ_kFSXRXoE0Wo74JlSY/edit (is that all?)

# Native Histograms

Native histograms were introduced as an experimental feature in November 2022.
They are a concept that touches almost every part of the Prometheus stack. The
first version of the Prometheus server supporting native histograms was
v2.40.0. The support had to be enabled via a feature flag
`--enable-feature=native-histograms`. (TODO: This is still the case with the
current release v2.49. Update this section with the stable release, once it has
happened.)

Due to the pervasive nature of the native-histogram related changes, the
documentation of those changes and explanation of the underlying concepts are
widely distributed over various channels (like the documentation of affected
Prometheus components, doc comments in source code, sometimes the source code
itself, design docs, conference talks, …). This document intends to gather all
these pieces of information and present them concisely in a unified context.
This document prefers to link existing detailed documentation rather than
restating it, but it contains enough information to be comprehensible without
referring to other sources.

While formal specifications are supposed to happen in their respective context
(e.g. OpenMetrics changes will be specified in the general OpenMetrics
specification), some parts of this document take the shape of a specification.
In those parts, the key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL
NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” are used as
described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

The key word TODO may refer to either of the following:
- Incomplete parts of this document.
- Incomplete implementation of intended native-histogram related changes.

## Introduction

The core idea of native histograms is to treat histograms as first class
citizens in the Prometheus data model. This approach unlocks the features
described in the following, which is the reason why they are called _native
histograms_.

Previously, all Prometheus sample values had been 64-bit floating point values
(short _float64_ or just _float_). These floats can directly represent _gauges_
or _counters_. The Prometheus metric types _summary_ and _histogram_, as they
exist in exposition formats, are broken down into float components upon
ingestion: A _sum_ and a _count_ component for both types, a number of
_quantile_ samples for a summary and a number of _bucket_ samples for a
histogram.

With native histograms, a new structured sample type is introduced. A single
sample represents the previously known _sum_ and _count_ plus a dynamic set of
buckets.

Native histograms have the following key properties:
1. A sparse bucket representation, allowing (near) zero cost for empty buckets.
2. Coverage of the full float64 range of values.
3. No configuration of bucket boundaries during instrumentation.
4. Dynamic resolution picked according to simple configuration parameters.
5. A sophisticated exponential bucketing schema, ensuring mergeability between
   all histograms.
6. An efficient data representation for both exposition and storage.

Compared to the previously existing “classic” histograms, native histograms
allow a higher bucket resolution across arbitrary ranges of observed values at
a lower storage and query cost with very little to no configuration required.
Even partitioning histograms by labels is now much more affordable.

Because the sparse representation (property 1 in the list above) is so crucial
for many of the other benefits of native histograms that_sparse histograms_ was
a common name for _native histograms_ early during the design process. However,
other key properties like the exponential bucketing schema or the dynamic
nature of the buckets are also very important, but not caught at all in the
term _sparse histograms_.

### Design docs

These are the design docs that guided the development of native histograms.
Some details are obsolete now, but they describe rather well the underlying
concepts and how they evolved.

- [Sparse high-resolution histograms for
  Prometheus](https://docs.google.com/document/d/1cLNv3aufPZb3fNfaJgdaRBZsInZKKIHo9E6HinJVbpM/edit),
  the original design doc.
- [Prometheus Sparse Histograms and
  PromQL](https://docs.google.com/document/d/1ch6ru8GKg03N02jRjYriurt-CZqUVY09evPg6yKTA1s/edit),
  more an exploratory document than a proper design doc about the handling of
  native histograms in PromQL.

### Conference talks

A more approachable way of learning about native histograms is to watch
conference talks, of which a selection is presented below. As an introduction,
it might make sense to watch these talks and then return to this document to
learn about all the details and technicalities.

- [Secret History of Prometheus
  Histograms](https://fosdem.org/2020/schedule/event/histograms/) about the
  classic histograms and why Prometheus kept them for so long.
- [Prometheus Histograms – Past, Present, and
  Future](https://promcon.io/2019-munich/talks/prometheus-histograms-past-present-and-future/)
  is the inaugural talk about the new approach that lead to native histograms.
- [Better Histograms for
  Prometheus](https://www.youtube.com/watch?v=HG7uzON-IDM) explains why the
  concepts work out in practice.
- [Native Histograms in
  Prometheus](https://promcon.io/2022-munich/talks/native-histograms-in-prometheus/)
  presents and explains native histograms after the actual implementation.
- [PromQL for Native
  Histograms](https://promcon.io/2022-munich/talks/promql-for-native-histograms/)
  explains the usage of native histograms in PromQL.
- [Prometheus Native Histograms in
  Production](https://www.youtube.com/watch?v=TgINvIK9SYc) provides an analysis
  of performance and resource consumption.
- [Using OpenTelemetry’s Exponential Histograms in
  Prometheus](https://www.youtube.com/watch?v=W2_TpDcess8) covers the
  interoperability with the OpenTelemetry.

## Glossary

- A __native histogram__ is the new complex sample type representing a full
  histogram that this document is about.
- A __classic histogram__ is the older sample type representing a histogram
  with fixed buckets, formerly just called a _histogram_. It exists as such in
  the exposition formats, but is broken into a number of float samples upon
  ingestion into Prometheus.
- __Sparse histogram__ is an older, now deprecated name for _native
  histogram_. This name might still be found occasionally in older
  documentation. __Sparse buckets__ remains a meaningful term for the buckets
  of a native histogram.

## Data model

This section describes the data model of native histograms in general. It
avoids implementation specifics as far as possible. This includes terminology.
For example, a _list_ described in this section will become a _repeated
message_ in a protobuf implementation and (most likely) a _slice_ in a Go
implementation.

### General structure

Similar to a classic histogram, a native histogram has a field for the _count_
of observations and a field for the _sum_ of observations. In addition, it
contains the following components, which are described in detail in dedicated
sections below:

- A _schema_ to identify the method of determining the boundaries of any given
  bucket with an index _i_.
- A sparse representation of indexed buckets, mirrored for positive and
  negative observations.
- A _zero bucket_ to count observations close to zero.
- A list of _custom values_.
- _Exemplars_.

### Flavors

Any native histogram has a specific flavor along each of two independent
dimensions:

1. Counter vs. gauge: Usually, a histogram is “counter like”, i.e. each of its
   buckets acts as a counter of observations. However, there are also “gauge
   like” histograms where each bucket is a gauge, representing arbitrary
   distributions at a point in time. The concept was previously introduced for
   classic histograms by
   [OpenMetrics](https://github.com/OpenObservability/OpenMetrics/blob/main/specification/OpenMetrics.md#gaugehistogram).
2. Integer vs. floating point (short: float): The obvious use case of
   histograms is to count observations, resulting in positive integer numbers
   of observations within each bucket, including the _zero bucket_, and for the
   total _count_ of observations, represented as unsigned 64-bit integers
   (short: uint64). However, there are specific use cases leading to a
   “weighted” or “scaled” histogram, where all of these values are represented
   as 64-bit floating point numbers (short: float64). Note that the _sum_ of
   observations is a float64 in either case.

Float histograms are occasionally used in direct instrumentation for “weighted”
observations, for example to count the number of seconds an observed value was
falling into different buckets of a histogram. The for more common use case for
float histograms is within PromQL, though. PromQL generally only acts on float
values, so the PromQL engine converts every histogram retrieved from the TSDB
to a float histogram first, and any histogram stored back into TSDB via
recording rules is a float histogram. If such a histogram is effectively an
integer histogram (because all non-_sum_ fields are float numbers that
represent integer values), a TSDB implementation MAY convert them back to
integer histograms to increase storage efficiency. (As of Prometheus v2.51, the
TSDB implementation within Prometheus is not utilizing this option.) Note,
however, that the most common PromQL function applied to a counter histogram is
`rate`, which generally produces non-integer numbers, so that results of
recording rules will commonly be float histograms with non-integer values
anyway.

Treating native histograms explicitly as integer histograms vs. float histogram
is a notable deviation from the treatment of conventional simple numeric
samples, which are always treated as floats throughout the whole stack for the
sake of simplicity.

The main reason for the more involved treatment of histograms is the easy
efficiency gains in protobuf-based exposition formats. Protobuf uses varint
encoding for integers, which reduces the data size for small integer values
without requiring an additional compression layer. This benefit is amplified by
the [delta encoding of integer buckets](#buckets), which generally results in
smaller integer values. Floats, in contrast, always require 8 bytes in
protobuf. In practice, many integers in an integer histogram will fit in 1
byte, and most will fit in 2 bytes, so that the explicit presence of integer
histogram in a protobuf-exposition format will directly result in a data size
reduction approaching 8x for histograms with many buckets. This is particularly
relevant as the overwhelming majority of histograms exposed by instrumented
targets are integer histograms.

For similar reasons, representing integer histograms in RAM and on disk is
generally more efficient than float histograms. This is less relevant than the
benefits in the exposition format, though. For one, Prometheus uses
Gorilla-style XOR encoding for floats, which reduces their size, albeit not as
much as the double-delta encoding used for integers. More importantly, an
implementation could always decide to internally use an integer representation
for histogram fields that are effectively integer values (see above).
(Historical note: Prometheus v1 used exactly this approach to improve the
compression of float samples, and Prometheus v2 might very well adopt this
approach again in the future.)

In a counter histogram, the total _count_ of observation and the counts in the
buckets individually behave as Prometheus counters, i.e. they only go down upon
a counter reset. However, the _sum_ of observation may decrease as a
consequence of the observation of negative values. PromQL implementations MUST
detect counter resets based on the whole histogram (see [counter reset
considerations section](#counter-reset-considerations) below for details).
(Note that this always has been a problem for the _sum_ component of classic
histograms and summaries, too. The approach so far was to accept that counter
reset detection silently breaks for _sum_ in those cases. Fortunately, negative
observations are a very rare use case for Prometheus histograms and summaries.)

### Schema

The _schema_ is a signed integer value with a size of 8 bits (short: int8). It
defines the way bucket boundaries are calculated. The currently valid values
are -53 and the range between and including -4 and +8. More schemas may be
added in the future. -53 is a schema for so-called _custom buckets_, while the
other schema numbers represent the different standard exponential schemas
(short: _standard schemas_).

The standard schemas are mergeable with each other and are RECOMMENDED for
general use cases. Larger schema numbers correspond to higher resolutions.
Schema _n_ has half the resolution of schema _n_+1, which implies that a
histogram with schema _n_+1 can be converted into a histogram with schema _n_
by merging neighboring buckets.

For any standard schema _n_, the boundaries of a bucket with index _i_
calculated as follows (using Python syntax):

- The upper inclusive limit of a positive bucket:
  `(2**2**-n)**i`
- The lower exclusive limit of a positive bucket:
  `(2**2**-n)**(i-1)`
- The lower inclusive limit of a negative bucket:
  `-((2**2**-n)**i)`
- The upper exclusive limit of a negative bucket:
  `-((2**2**-n)**(i-1))`

_i_ is an integer number that may be negative.

There are exceptions to the rules above concerning the largest and smallest
finite values representable as a float64 (called `MaxFloat64` and `MinFloat64`
in the following) and the positive and negative infinity values (`+Inf` and
`-Inf`):

- The positive bucket that contains `MaxFloat64` has an upper inclusive limit
  of `MaxFloat64`.
- The next positive bucket (index _i_+1 relative to the bucket from the
  previous item) has a lower exclusive limit of `MaxFloat64` and an upper
  inclusive limit of `+Inf`. (It could be called a _positive overflow bucket_.)
- The negative bucket that contains `MinFloat64` has a lower inclusive limit
  of `MinFloat64`.
- The next negative bucket (index _i_+1 relative to the bucket from the
  previous item) has an upper exclusive limit of `MinFloat64` and an lower
  inclusive limit of `-Inf`. (It could be called a _negative overflow bucket_.)
- Buckets beyond the `+Inf` and `-Inf` buckets described above MUST NOT be used.
  
There are more exceptions for values close to zero, see the [zero bucket
section](#zero-bucket) below.

The current limits of -4 for the lowest resolution and 8 for the highest
resolution have been chosen based on practical usefulness. Should a practical
need arise for even lower or higher resolution, an extension of the range will
be considered. However, a schema greater than 52 does not make sense as the
growth factor from one bucket to the next would then be smaller than the
difference between representable float64 numbers. Likewise, a schema smaller
than -9 does not make sense either, as the growth factor would then exceed the
largest float representable as float64. Therefore, the schema numbers between
(and including) -9 and +52 are reserved for future standard schemas (following
the formulas for bucket boundaries above) and MUST NOT be used for any other
schemas.

For schema -53, the bucket boundaries are set explicitly via _custom values_,
described in detail in the [custom values section](#custom-values) below. This
results in a native histogram with custom buckets (often abbreviated as NHCB).
Such a histogram can be used to represent a classic histogram as a native
histogram. It could also be used if the exponential bucketing featured by the
standard schemas is a bad match for the distribution to be represented by the
histogram. Histograms with different custom bucket boundaries are generally not
mergeable with each other. Therefore, schema -53 SHOULD only be used as an
informed decision in specific use cases.

### Buckets

For standard schemas, buckets are represented as two lists, one for positive
buckets and one for negative buckets. For custom buckets (schema -53), only the
positive bucket list is used, but repurposed for all buckets.

Any unpopulated buckets MAY be excluded from the lists. (Which is the reason
why the buckets are often called _sparse buckets_.)

For float histograms, the elements of the lists are float64 and
represent the bucket population directly.

For integer histograms, the elements of the lists are signed 64-bit integers
(short: int64), and each element represents the bucket population as a delta to
the previous bucket in the list. The first bucket in the list contains an
absolute population (i.e. a delta relative to zero).

To map buckets in the lists to the indices as defined in the previous section,
there are two lists of so-called _spans_, one for the positive buckets and one
for the negative buckets.

Each span consists of a pair of numbers, a signed 32-bit integer (short: int32)
called _offset_ and an unsigned 32-bit integer (short: uint32) called _length_.
Only the first span in each list can have a negative offset. It defines the
index of the first bucket in its corresponding bucket list. The length defines
the number of consecutive buckets the bucket list starts with. The offsets of
the following spans define the number of excluded (and thus unpopulated
buckets). The lengths define the number of consecutive buckets in the list
following the excluded buckets.

The sum of all lengths in each span list MUST be equal to the length of the
corresponding bucket list.

Empty spans (with a length of zero) are valid and MAY be used, although they
are generally not useful and they SHOULD be eliminated by adding their offset
to the offset of the following span. Similarly, spans that are not the first
span in a list MAY have an offset of zero, although those offsets SHOULD be
eliminated by adding their length to the previous span. Both cases are allowed
so that producers of native histograms MAY pick whatever representation has the
best resource trade-offs at that moment. For example, if a histogram is
processed through various stages, it might be most efficient to only eliminate
redundant spans after the last processing stage.

In a similar spirit, there are situation where excluding every unpopulated
bucket from the bucket list is most efficient, but in other situations, it
might be better to reduce the number of spans by representing small numbers of
unpopulated buckets explicitly.

Note that future high resolution schemas might require offsets that are too
large to be represented with an int32. An extension of the data model will be
required in that case. (The current standard schema with the highest resolution
is schema 8, for which the bucket that contains `MaxFloat64` has index 262144,
and thus the `+Inf` overflow bucket has index 262145, while the largest number
representable with int32 is 2147483647. The highest standard schema that would
still work with int32 offsets would be schema 20, corresponding to a growth
factor from bucket to bucket of only ~1.000000661.)

#### Example

An integer histogram has the following positive buckets (index→population):

`-2→3, -1→5, 0→0, 1→0, 2→1, 3→0, 4→3, 5→2`

They could be represented in this way:

- Positive bucket list: `[3, 2, -4, 2, -1]`
- Positive span list: `[[-2, 2], [2,1], [1,2]]`

The second and third span could be merged into one if the single unpopulated
bucket with index 3 is represented explicitly, leading to the following result:

- Positive bucket list: `[3, 2, -4, -1, 3, -1]`
- Positive span list: `[[-2, 2], [2,4]]`

Or merge all the spans into one by representing all unpopulated buckets above
explicitly:

- Positive bucket list: `[3, 2, -5, 0, 1, -1, 3, -1]`
- Positive span list: `[[-2, 8]]`

### Zero bucket

Observations of exactly zero do not fit into any bucket as defined by the
standard schemas above. They go to a dedicated bucket called the _zero bucket_.

The number of observations in the zero bucket is tracked by a single uint64
(for integer histograms) or float64 (for float histograms).

The zero bucket has an additional parameter called the _width_ of the zero
bucket, which is a float64 ≥ 0. If the width is set to zero, only observations
of exactly zero go to the zero bucket, which is the case described above. If
the width has a positive value, all observations within the closed interval
[-width, +width] go to the zero bucket rather than a regular bucket. This has
two use cases:

- Noisy observations close to zero tend to populate a high number of buckets.
  Those observations might happen due to numerical inaccuracies or if the
  source of the observations are actual physical measurements. A zero bucket
  with a relatively small width redirects those observations into a single
  bucket.
- If the user is more interested in the long tail of a distribution, far away
  from zero, a relatively large width of the zero bucket helps to avoid many
  high resolution buckets for a range that is not of interest.

The width of a zero bucket SHOULD coincide with a boundary of a regular bucket,
which avoids the complication of the zero bucket overlapping with parts of a
regular bucket. However, if such an overlap is happening, the observations that
are counted in the regular bucket overlapping with the zero bucket MUST be
outside of the [-width, +width] interval.

To merge histograms with the same zero bucket width, the two zero buckets
are simply added. If the zero bucket widths in the source histograms are
different, however, the largest width in any of the source histograms is
chosen. If that width happens to be within any populated bucket in the other
source histograms, the width is increased until one of the following is true
for each source histogram:

- The new width coincides with the boundary of a populated bucket.
- The new width is not within any populated bucket.

Then the source zero buckets and any source buckets now inside the new width are
added up to yield the population of the new zero bucket.

The zero bucket is not used if the schema is -53 (custom buckets).

### Custom values

The list of custom values is unused for standard schemas. The only currently
defined schema for which custom values are used is -53 (custom buckets).

// Explain.

### Exemplars

// MUST have timestamp

### Special cases of observed values

// Overflow, Inf, NaNs.

### OpenTelemetry interoperability

## Exposition formats

### Classic Prometheus formats

// Only proto
// Combined classic/native, how to mark observation-less native histograms
// MAY take exemplars from classic buckets.
// Heuristics for exemplars.

### OpenMetrics

## Instrumentation libraries

// Link to library doc instead of replicating.
// Here is more for general guidelines.
// Both usage as well as implementation details, e.g. observing NaNs (maybe
already explained sufficinetly enough?).
// bucket limitation strategies
// partition by labels at will, why feasible
// OTel approach: "infinite" resolution, going down
// Default values

## Scrape configuration

To enable the Prometheus server to scrape native histograms, the feature flag
`--enable-feature=native-histograms` is required. This flag also changes the
content negotiation to prefer the classic protobuf-based exposition format over
the OpenMetrics text format. (TODO: This behavior will change once native
histograms are a stable feature.)

// TODO: how to configure NHCB conversion from classic histograms.

### Finetuning content negotiation

### Limit bucket count and resolution

### Scrape both classic and native histograms

## TSDB

// Mixed series.
// Staleness. Equivalence with float.
// When to cut a new chunk.

### Counter reset considerations

### Out-of-order support

## PromQL

// always float (but MAY re-convert to int for storage, remote-write)
// merging
// counter reset handling
// including for _sum_
// interpolation behavior, including zero bucket and NHCB handling

### Recording rules

### Alerting rules

### Testing framework

## Prometheus query API

## Prometheus UI

// Note that Grafana support etc. is out of scope.

## Template expansion

## Remote write & read

// Include NHCB conversion from classic histograms?

## Pushgateway

## `promtool` 

This section describes `promtool` commands added or changed to support native
histograms.

## `prom2json`

## Migration considerations

// Scraping.
// Querying.
// Storing classic histograms as native histograms.

## Future extensions

### Custom bucket layouts

