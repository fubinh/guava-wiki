# Computing statistical values


This page lists a number of common statistical computations and how to perform
them, often making use of the statistical support libraries in
`com.google.common.math`.

In the following examples, a variable with a name like `intArray` or
`collectionOfDouble` is of the type implied by that name. The identifier
`values` can represent an `int[]`, `long[]`, `double[]`, `Collection<? extends
Number>`, or can be replaced with primitive varargs. (In some cases, even more
variations may be accepted; check Javadoc for full details.)

Links to named classes are given at the [bottom](#links) of the page.


<a name="mean"></a>
## Mean (only) of existing values

```java
double mean = Stats.meanOf(values);

double mean = doubleStream.average().getAsDouble();
```

<a name="max"></a>
## Maximum (only) of existing values

```java
double max = doubleStream.max().getAsDouble();

double max = Doubles.max(doubleArray);

double max = immutableDoubleArray.stream().max().getAsDouble();

double max = Collections.max(collectionOfDouble);

double max = Ordering.natural().max(iterableOfDouble);
```

<a name="sum"></a>
## Sum (only) of existing values

```java
double sum = doubleStream.sum();

double sum = Arrays.stream(doubleArray).sum();

double sum = Stats.of(values).sum();
```

<a name="mean_max"></a>
## Both mean and maximum of existing values

```java
DoubleSummaryStatistics stats = doubleStream.summaryStatistics();
double mean = stats.getAverage();
double max = stats.getMax();

Stats stats = Stats.of(values);
double mean = stats.mean();
double max = stats.max();
```

<a name="stddev"></a>
## Standard deviation of existing values

Choose between `populationStandardDeviation` and `sampleStandardDeviation`; see
the Javadoc of these methods to understand the difference. You can get other
statistics, such as mean, min, and max, from the same `Stats` instance.

```java
double stddev = Stats.of(values).populationStandardDeviation();

double stddev = primitiveStream.collect(toStats()).populationStandardDeviation();
```

(The `toStats()` method is statically imported from `Stats`.)

<a name="accumulate"></a>
## Mean and sample standard deviation of incoming values

This approach is useful when you don't want to store up all the values in
advance. Instead, create an "acccumulator", and as you get the values you can
feed them in and then discard them.

```java
StatsAccumulator accum = new StatsAccumulator();
...

// any number of times, over time
accum.add(value); // or addAll
...

double mean = accum.mean();
double stddev = accum.sampleStandardDeviation();

// or use accum.snapshot() to get an immutable Stats instance
```

<a name="median"></a>
## Median (only) of existing values

```java
double median = Quantiles.median().compute(values);
```

<a name="percentile"></a>
## 95th percentile of existing values

```java
double percentile95 = Quantiles.percentiles().index(95).compute(values);
```

<a name="percentiles"></a>
## Find the 90th, 99th, and 99.9th percentile

```java
Map<Integer, Double> largeValues =
    Quantiles.scale(1000).indexes(900, 990, 999).compute(values);
double p99 = largeValues.get(990); // for example
```

<a name="correlation"></a>
## Find the statistical correlation between two sets of values

```java
PairedStatsAccumulator accum = new PairedStatsAccumulator();

for (...) {
  ...
  accum.add(x, y);
}

double correl = accum.pearsonsCorrelationCoefficient();
```

<a name="linear"></a>
## Find a linear approximation for a set of ordered pairs

```java
PairedStatsAccumulator accum = new PairedStatsAccumulator();

for (...) {
  ...
  accum.add(x, y);
}

LinearTransformation bestFit = accum.leastSquaresFit();
double slope = bestFit.slope();
double yIntercept = bestFit.transform(0);
double estimateXWhenYEquals5 = bestFit.inverse().transform(5);
```

<a name="links"></a>
## Links to classes used in these examples

*   [`Arrays`]
*   [`Collections`]
*   [`Doubles`]
*   [`DoubleStream`]
*   [`DoubleSummaryStatistics`]
*   [`ImmutableIntArray`]
*   [`LinearTransformation`]
*   [`Longs`]
*   [`Ordering`]
*   [`PairedStatsAccumulator`]
*   [`Quantiles`]
*   [`Stats`]
*   [`StatsAccumulator`]
*   [`Streams`]

[`Arrays`]: https://docs.oracle.com/javase/8/docs/api/java/util/Arrays.html
[`Collections`]: https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html

[`Doubles`]: http://google.github.io/guava/releases/snapshot-jre/api/docs/com/google/common/primitives/Doubles.html

[`DoubleStream`]: https://docs.oracle.com/javase/8/docs/api/java/util/stream/DoubleStream.html
[`DoubleSummaryStatistics`]: https://docs.oracle.com/javase/8/docs/api/java/util/DoubleSummaryStatistics.html

[`ImmutableIntArray`]: http://google.github.io/guava/releases/snapshot-jre/api/docs/com/google/common/primitives/ImmutableIntArray.html
[`LinearTransformation`]: http://google.github.io/guava/releases/snapshot-jre/api/docs/com/google/common/math/LinearTransformation.html
[`Longs`]: http://google.github.io/guava/releases/snapshot-jre/api/docs/com/google/common/primitives/Longs.html
[`Ordering`]: http://google.github.io/guava/releases/snapshot-jre/api/docs/com/google/common/collect/Ordering.html
[`PairedStatsAccumulator`]: http://google.github.io/guava/releases/snapshot-jre/api/docs/com/google/common/math/PairedStatsAccumulator.html
[`Quantiles`]: http://google.github.io/guava/releases/snapshot-jre/api/docs/com/google/common/math/Quantiles.html
[`Stats`]: http://google.github.io/guava/releases/snapshot-jre/api/docs/com/google/common/math/Stats.html
[`StatsAccumulator`]: http://google.github.io/guava/releases/snapshot-jre/api/docs/com/google/common/math/StatsAccumulator.html
[`Streams`]: http://google.github.io/guava/releases/snapshot-jre/api/docs/com/google/common/collect/Streams.html
