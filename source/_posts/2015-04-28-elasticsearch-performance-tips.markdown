---
layout: post
title: "ElasticSearch Performance Tips"
date: 2015-04-28 23:08
categories: [Notes]
published: false
---

Recently we're using ElasticSearch as a data backend of our recommendation API, to serve both offline and online computed data to users. Thanks to ElasticSearch's rich and out-of-the-box functionality, it doesn't take much trouble to setup the cluster. However, we also encounter some misuse and unwise configurations. So here's a list of ElasticSearch performance tips that we learned from practice.

## Tip#1 Set Num-of-shards to Num-of-nodes

<!-- more -->

## Tip#2 Disable Unnecessary Features

## Tip#3 Use Bulk Operations Whenever Is Possible

## Tip#4 Tuning Memory Usage

## Tip#5 Setup a Cluster with Unicast

## References
