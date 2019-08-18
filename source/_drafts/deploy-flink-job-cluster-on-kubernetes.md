---
title: Deploy Flink Job Cluster on Kubernetes
categories: Big Data
tags: [kubernetes, flink]
---

[Kubernetes][8] is the trending container orchestration system that can be used to host various applications from web services to data processing jobs. Applications are packaged in self-contained, yet light-weight containers, and we declare how they should be deployed, how they scale, and how they expose as services. [Flink][9] is also a trending distributed computing framework that can run on a variety of platforms, including Kubernetes. Combining them will bring us robust and scalable deployments of data processing jobs, and more safely Flink can share a Kubernetes cluster with other services.

When deploying Flink on Kubernetes, there are two options, session cluster and job cluster. TODO

![Flink on Kubernetes](/blog/images/flink-on-k8s.png)

Here are the steps of running a Flink job cluster on Kubernetes:

* Compile and package the Flink job jar.
* Build a Docker image containing the Flink runtime and the job jar.
* Create a Kubernetes Job for Flink JobManager.
* Create a Kubernetes Service for this Job.
* Create a Kubernetes Deployment for Flink TaskManagers.
* Enable Flink JobManager HA with ZooKeeper.
* Correctly stop and resume Flink job with SavePoint facility.

<!-- more -->

## Kubernetes Playground

In case you do not already have a Kubernetes environment, one can easily setup a local playground with [minikube][1]. Take MacOS for example:

* Install [VirtualBox][2], since minikube will setup a k8s cluster inside a virtual machine.
* Download the [minikube binary][3], making it executable and accessible from PATH.
* Execute `minikube start`, it will download the virtual machine image, kubelet and kubeadm facilities, install and verify the k8s cluster. If you have trouble accessing the internet, setup a proxy and [tell minikube to use it][5].
* Download and install the [kubectl binary][4]. Minikube has configured kubectl to point to the installed k8s cluster, so one can execute `kubectl get pods -A` to see the running system pods.

```text
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   kube-apiserver-minikube            1/1     Running   0          16m
kube-system   etcd-minikube                      1/1     Running   0          15m
kube-system   coredns-5c98db65d4-d4t2h           1/1     Running   0          17m
```

## Flink Streaming Job

Let us create a simple streaming job, that reads data from socket, and prints the count of words every 5 seconds. The following code is taken from [Flink doc][6], and a full Maven project can be found on [GitHub][7].

```java
DataStream<Tuple2<String, Integer>> dataStream = env
    .socketTextStream("localhost", 9999)
    .flatMap(new Splitter())
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1);

dataStream.print();
```

Run `mvn clean package`, and the compiled job jar can be found in `target/flink-on-kubernetes-jar-with-dependencies.jar`.

## Build Docker Image

## Deploy Job Cluster

### JobManager

### Task Manager

## Configure JobManager HA

## Manage Flink Job

### Stop and Resume

### Scale

## Logs and Monitoring

## References

* https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/deployment/kubernetes.html
* https://github.com/apache/flink/blob/release-1.8/flink-container/docker/README.md
* https://github.com/apache/flink/blob/release-1.8/flink-container/kubernetes/README.md#deploy-flink-job-cluster
* https://github.com/docker-flink/docker-flink/blob/master/1.8/scala_2.12-debian/Dockerfile
* https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/
* https://jobs.zalando.com/tech/blog/running-apache-flink-on-kubernetes/
* https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=65147077

[1]: https://github.com/kubernetes/minikube
[2]: https://www.virtualbox.org
[3]: https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
[4]: https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/darwin/amd64/kubectl
[5]: https://minikube.sigs.k8s.io/docs/reference/networking/proxy/
[6]: https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/datastream_api.html#example-program
[7]: https://github.com/jizhang/flink-on-kubernetes
[8]: https://kubernetes.io/
[9]: https://flink.apache.org/
