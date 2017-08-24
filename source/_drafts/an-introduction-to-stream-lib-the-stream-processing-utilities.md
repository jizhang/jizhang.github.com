---
title: An Introduction to stream-lib The Stream Processing Utilities
tags: [stream processing, java, algorithm]
categories: [Big Data]
---

<!-- more -->

## Count Cardinality with `HyperLogLog`

```java

```

* use case: uv
* cardinalities of > 10^9, error rate 2%, 1.5KB memory
* algorithm
  * maximum number of leading zeros n, number of distinct values 2^n
  * hash function to obtain uniformly distributed random numbers
  * solve variance - split into subsets and combine by harmonic mean
* process
  * use first *b* bits to decide which register to modify
  * compute leading zeros in the remaining bits, and update register if it's greater
  * harmonic mean of all registers
* hll++
  * 64 bit hash function to remove alpha correction
  * small cardinality, empirical bias correction
  * sparse grows to dense

https://en.wikipedia.org/wiki/HyperLogLog#HLL.2B.2B

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

* https://github.com/addthis/stream-lib
* https://www.javadoc.io/doc/com.clearspring.analytics/stream/2.9.5
* http://www.addthis.com/blog/2011/03/29/new-open-source-stream-summarizing-java-library/
* https://www.mapr.com/blog/some-important-streaming-algorithms-you-should-know-about
