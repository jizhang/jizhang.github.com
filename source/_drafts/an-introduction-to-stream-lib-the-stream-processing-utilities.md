---
title: An Introduction to stream-lib The Stream Processing Utilities
tags: [stream processing, java, algorithm]
categories: [Big Data]
---

When processing a large amount of data, certain operations will cost a lot of time and space, such as counting the distinct values, or figuring out the 95th percentile of a sequence of numbers. But sometimes the accuracy is not that important. Maybe you just want a brief summary of the dataset, or it's a monitoring system, where limited error rate is tolerable. There're plenty of such algorithms that can trade accuracy with huge saves of time-space. What's more, most of the data structures can be merged, making it possible to use in stream processing applications. [`stream-lib`][1] is a collection of these algorithms. They are Java implementations based on academical research and papers. This artile will give a brief introduction to this utility library.

## Count Cardinality with `HyperLogLog`

Unique visitors (UV) is the major metric of websites. We usually generate UUIDs for each user and track them by HTTP Cookie, or roughly use the IP address. We can use a `HashSet` to count the exact value of UV, but that takes a lot of memory. With `HyperLogLog`, an algorithm for the count-distinct problem, we are able to [estimate cardinalities of > 10^9 with a typical accuracy of 2%, using 1.5 kB of memory][2].

```xml
<dependency>
    <groupId>com.clearspring.analytics</groupId>
    <artifactId>stream</artifactId>
    <version>2.9.5</version>
</dependency>
```

```java
ICardinality card = new HyperLogLog(10);
for (int i : new int[] { 1, 2, 3, 2, 4, 3 }) {
    card.offer(i);
}
System.out.println(card.cardinality()); // output: 4
```

<!-- more -->

`HyperLogLog` estimates cardinality by counting the leading zeros of each member's binary value. If the maximum count is *n*, the cardinality is *2^n*. There're some key points in this algorithm. First, members needs to be uniformly distributed, which we can use a hash function to achieve. `stream-lib` uses [MurmurHash][3], a simple, fast, and well distributed hash function, that is used in lots of hash-based lookup algorithms. Second, to decrease the variance of the result, set members are splitted into subsets, and the final result is the harmonic mean of all subsets' cardinality. The integer argument that we passed to `HyperLogLog` constructor is the number of bits that it'll use to split subsets, and the accuracy can be derived from this formula: `1.04/sqrt(2^log2m)`.

`HyperLogLog` is an extension of `LogLog` algorithm, and the `HyperLogLogPlus` makes some more improvements. For instance, it uses a 64 bit hash function to remove the correction factor that adjusts hash collision; for small cardinality, it applies an empirical bias correction; and it also supports growing from a sparse data strucutre of registers (holding subsets) to a dense one. These algorithms are all included in `stream-lib`

## Test Membership with `BloomFilter`

```java

```

* construction process
  * an array of n bits
  * apply k hash functions and set the bits
  * when testing, apply k hash functions
    * if every bit hits, it might be in the set, with a False Position Probability
    * if not all, it must not be in the set.
* choice of hash function
  * independent and uniformly distributed
  * fast
* FPP
  * number of bits (m), functions (k),
  * (1 - e^(-kn/m))^k
* use case
  * spam emails
  * malicious urls in Chrome
  * Cassandra, HBase, non-existent rows or columns to reduce disk lookups
  * Squid cache digest

http://spyced.blogspot.com/2009/01/all-you-ever-wanted-to-know-about.html
http://llimllib.github.io/bloomfilter-tutorial/
https://en.wikipedia.org/wiki/Bloom_filter#Examples


## Top-k Elements with `CountMinSketch`

```java
// priority queue
```

* estimate the frequency of item
* hash table takes O(n) space
* algorithm
  * number of hash functions *d*, arrays size *w*
  * calculate hash values for incoming item, and save to the `d * w` matrix;
  * when *point query*  the frequency of some item, calc hash values and get the smallest;
* accuracy:
  * estimation error: `ε = e / w` - increase w to increase accuracy of results;
  * probability of bad estimate: `δ = 1 / e ^ d` - increase d to decrease prop;
  * it's always overestimate
  * optimal for skewed values
* hash functions - [pairwise independent](https://en.wikipedia.org/wiki/Pairwise_independence)

* alternative: `StreamSummary`

https://stackoverflow.com/a/35356116/1030720
https://highlyscalable.wordpress.com/2012/05/01/probabilistic-structures-web-analytics-data-mining/
paper: https://web.archive.org/web/20060907232042/http://www.eecs.harvard.edu/~michaelm/CS222/countmin.pdf
http://quangpham-cs100w.blogspot.com/2013/11/count-min-sketch-data-structure.html
https://en.wikipedia.org/wiki/Count%E2%80%93min_sketch

## Histogram and Quantile with `TDigest`

* use case: median, 95% percentile
* algorithm:
  * a variant of 1-dimensinal k-means clustering
  * represents the empirical distribution of a set of numbers by retaining centroids of sub-sets of numbers.
  * bin size is not fixed
  * merge bins when the meet size criterion
* parallel friendly - merge and become more accurate

https://github.com/tdunning/t-digest
https://dataorigami.net/blogs/napkin-folding/19055451-percentile-and-quantile-estimation-of-big-data-the-t-digest
paper: https://raw.githubusercontent.com/tdunning/t-digest/master/docs/t-digest-paper/histo.pdf

## References

* https://www.javadoc.io/doc/com.clearspring.analytics/stream/2.9.5
* http://www.addthis.com/blog/2011/03/29/new-open-source-stream-summarizing-java-library/
* https://www.mapr.com/blog/some-important-streaming-algorithms-you-should-know-about

[1]: https://github.com/addthis/stream-lib
[2]: https://en.wikipedia.org/wiki/HyperLogLog
[3]: https://en.wikipedia.org/wiki/MurmurHash
