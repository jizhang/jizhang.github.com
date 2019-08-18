---
title: Deploy Flink Job Cluster on Kubernetes
categories: Big Data
tags: [kubernetes, flink]
---


![Flink on Kubernetes](/blog/images/flink-on-k8s.png)

Here are the steps of running a Flink job cluster on Kubernetes:

* Compile and package the Flink job jar.
* Build a Docker image containing the Flink runtime and the job jar.
* Create a Kubernetes Deployment for Flink TaskManagers.
* Create a Kubernetes Job for Flink JobManager.
* Create a Kubernetes Service for this Job.
* Enable Flink JobManager HA with ZooKeeper.
* Correctly stop and resume Flink job with SavePoint facility.

Let's go through these steps in details.

<!-- more -->

## Kubernetes Playground

## Flink Streaming Job

## Build Docker Image

## References

* https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/deployment/kubernetes.html
* https://github.com/apache/flink/blob/release-1.8/flink-container/docker/README.md
* https://github.com/apache/flink/blob/release-1.8/flink-container/kubernetes/README.md
* https://github.com/docker-flink/docker-flink/blob/master/1.8/scala_2.12-debian/Dockerfile
* https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/
