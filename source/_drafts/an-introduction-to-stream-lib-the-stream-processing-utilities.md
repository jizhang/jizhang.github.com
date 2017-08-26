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
System.out.println(card.cardinality()); // 4
```

<!-- more -->

`HyperLogLog` estimates cardinality by counting the leading zeros of each member's binary value. If the maximum count is `n`, the cardinality is `2^n`. There're some key points in this algorithm. First, members needs to be uniformly distributed, which we can use a hash function to achieve. `stream-lib` uses [MurmurHash][3], a simple, fast, and well distributed hash function, that is used in lots of hash-based lookup algorithms. Second, to decrease the variance of the result, set members are splitted into subsets, and the final result is the harmonic mean of all subsets' cardinality. The integer argument that we passed to `HyperLogLog` constructor is the number of bits that it'll use to split subsets, and the accuracy can be derived from this formula: `1.04/sqrt(2^log2m)`.

`HyperLogLog` is an extension of `LogLog` algorithm, and the `HyperLogLogPlus` makes some more improvements. For instance, it uses a 64 bit hash function to remove the correction factor that adjusts hash collision; for small cardinality, it applies an empirical bias correction; and it also supports growing from a sparse data strucutre of registers (holding subsets) to a dense one. These algorithms are all included in `stream-lib`

## Test Membership with `BloomFilter`

![Bloom Filter](/images/stream-lib/bloom-filter.jpg)

`BloomFilter` is a widely used data structure to test whether a set contains a certain member. The key is it will give false positive result, but never false negative. For example, Chrome maintains a malicious URLs in local storage, and it's a bloom filter. When typing a new URL, if the filter says it's not malicious, then it's definitely not. But if the filter says it is in the set, then Chrome needs to contact the remote server for further confirmation.

```java
Filter filter = new BloomFilter(100, 0.01);
filter.add("google.com");
filter.add("twitter.com");
filter.add("facebook.com");
System.out.println(filter.isPresent("bing.com")); // false
```

The contruction process of a bloom filter is faily simple:

1. Create a bit array of `n` bits. In Java, we can use the [`BitSet`][6] class.
2. Apply `k` number of hash functions to the incoming value, and set the corresponding bits to true.
3. When testing a membership, apply those hash functions and get the bits' values:
  * If every bit hits, the value might be in the set, with a False Positive Probability (FPP);
  * If not all bits hit, the value is definitely not in the set.

Again, those hash functions need to be uniformly distributed, and pairwise independent. Murmur hash meets the criteria. The FPP can be calculated by this formula: `(1-e^(-kn/m))^k`. This page ([link][4]) provides an online visualization of bloom filter. Other use cases are: anti-spam in email service, non-existent rows detection in Cassandra and HBase, and Squid also uses it to do [cache digest][5].

## Top-k Elements with `CountMinSketch`

![Count Min Sketch](/images/stream-lib/count-min-sketch.png)

[`CountMinSketch`][9] is a "sketching" algorithm that uses minimal space to track frequencies of incoming events. We can for example find out the top K tweets streaming out of Twitter, or count the most visited pages of a website. The "sketch" can be used to estimate these frequencies, with some loss of accuracy, of course.

The following snippet shows how to use `stream-lib` to get the top three animals in the `List`:

```java
List<String> animals;
IFrequency freq = new CountMinSketch(10, 10, 0);
Map<String, Long> top = Collections.emptyMap();
for (String animal : animals) {
    freq.add(animal, 1);
    top = Stream.concat(top.keySet().stream(), Stream.of(animal)).distinct()
              .map(a -> new SimpleEntry<String, Long>(a, freq.estimateCount(a)))
              .sorted(Comparator.comparing(SimpleEntry<String, Long>::getValue).reversed())
              .limit(3)
              .collect(Collectors.toMap(SimpleEntry::getKey, SimpleEntry::getValue));
}

System.out.println(top); // {rabbit=25, bird=45, spider=35}
```

`CountMinSketch#estimateCount` is a *point query* that asks for the count of an event. Since the "sketch" cannot remeber the exact events, we need to store them else where.

The data structure of count-min sketch is similar to bloom filter, instead of one bit array of `w` bits, it uses `d` number of them, so as to form a `d x w` matrix. When a value comes, it applies `d` number of hash functions, and update the corresponding bit in the matrix. These hash functions need only to be [pairwise independent][7], so `stream-lib` uses a simple yet fast `(a*x+b) mod p` formula. When doing *point query*, calculate the hash values, and the smallest value is the frequency.

The estimation error is `ε = e / w` while probability of bad estimate is `δ = 1 / e ^ d`. So we can increase `w` and / or decrease `d` to improve the results. Original paper can be found in this [link][8].

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
[4]: https://llimllib.github.io/bloomfilter-tutorial/
[5]: https://wiki.squid-cache.org/SquidFaq/CacheDigests
[6]: https://docs.oracle.com/javase/8/docs/api/java/util/BitSet.html
[7]: https://en.wikipedia.org/wiki/Pairwise_independence
[8]: https://web.archive.org/web/20060907232042/http://www.eecs.harvard.edu/~michaelm/CS222/countmin.pdf
[9]: https://stackoverflow.com/a/35356116/1030720
