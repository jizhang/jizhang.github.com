---
title: Build Interactive Report with Crossfilter and dc.js
tags: [crossfilter, dc.js, analytics]
categories: [Big Data]
---

When visualizing multidimensional datasets, we often want to connect individual charts together, so that one chart's filter will apply to all the other charts. We can do it manually, filter data on the server side, and update the rendered charts. Or we can filter data on the client side, and let charts update themselves. With Crossfilter and dc.js, this work becomes simple and intuitive.

## Airline On-time Performance

Here's an examples taken from Crossfilter's official website. It's a flight delay analysis report based on [ASA Data Expo](http://stat-computing.org/dataexpo/2009/) dataset. And this post will introduce how to use dc.js to build the report. A runnable JSFiddle can be found [here](https://jsfiddle.net/zjerryj/gjao9sws/8/), though the dataset is reduced to 1,000 records.

![](/images/airline-ontime-performance.png)

<!-- more -->

## Background

## Dataset, Dimension, and Measure

## Visualization

## Cross Filtering

## References

* [Crossfilter - Fast Multidimensional Filtering for Coordinated Views](http://crossfilter.github.io/crossfilter/)
* [dc.js - Dimensional Charting Javascript Library](https://dc-js.github.io/dc.js/)
* [Crossfiler Tutorial](http://blog.rusty.io/2012/09/17/crossfilter-tutorial/)
