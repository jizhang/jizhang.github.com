---
title: Deploy Flink Job Cluster on Kubernetes
categories: Big Data
tags: [kubernetes, flink]
---

[Kubernetes][8] is the trending container orchestration system that can be used to host various applications from web services to data processing jobs. Applications are packaged in self-contained, yet light-weight containers, and we declare how they should be deployed, how they scale, and how they expose as services. [Flink][9] is also a trending distributed computing framework that can run on a variety of platforms, including Kubernetes. Combining them will bring us robust and scalable deployments of data processing jobs, and more safely Flink can share a Kubernetes cluster with other services.

![Flink on Kubernetes](/images/flink-on-kubernetes.png)

When deploying Flink on Kubernetes, there are two options, session cluster and job cluster. Session cluster is like running a standalone Flink cluster on k8s that can accept multiple jobs and is suitable for short running tasks or ad-hoc queries. Job cluster, on the other hand, deploys a full set of Flink cluster for each individual job. We build container image for each job, and provide it with dedicated resources, so that jobs have less chance interfering with other, and can scale out independently. Besides, in future release, there might not be an option to run session cluster on k8s cluster ([FLIP-6][10]).

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

Run `mvn clean package`, and the compiled job jar can be found in `target/flink-on-kubernetes-0.0.1-SNAPSHOT-jar-with-dependencies.jar`.

## Build Docker Image

Flink provides an official docker image on [DockerHub][11]. We can use it as the base image and add job jar into it. Besides, in recent Flink distribution, the Hadoop binary is not included anymore, so we need to add Hadoop jar as well. Take a quick look of the base image's [Dockerfile][12], it does the following:

* Create from OpenJDK 1.8 base image.
* Install Flink in `/opt/flink`.
* Add `flink` user and group.
* Configure the entry point, which we will override in k8s deployments.

```Dockerfile
FROM openjdk:8-jre
ENV FLINK_HOME=/opt/flink
WORKDIR $FLINK_HOME
RUN useradd flink && \
  wget -O flink.tgz "$FLINK_TGZ_URL" && \
  tar -xf flink.tgz
ENTRYPOINT ["/docker-entrypoint.sh"]
```

Based on it, we create a new Dockerfile:

```Dockerfile
FROM flink:1.8.1-scala_2.12
ARG hadoop_jar
ARG job_jar
COPY --chown=flink:flink $hadoop_jar $job_jar $FLINK_HOME/lib/
```

Download the [Hadoop uber jar][12], and execute the following commands:

```bash
$ cd /path/to/Dockerfile
$ cp /path/to/flink-shaded-hadoop-2-uber-2.8.3-7.0.jar hadoop.jar
$ cp /path/to/flink-on-kubernetes-0.0.1-SNAPSHOT-jar-with-dependencies.jar job.jar
$ docker build --build-arg hadoop_jar=hadoop.jar --build-arg job_jar=job.jar --tag flink-on-kubernetes:0.0.1 .
```

Before building the image, you need to install Docker CLI and point it to the docker service inside minikube:

```bash
$ brew install docker
$ eval $(minikube docker-env)
```

Now we have a local docker image that is ready to be deployed.

```bash
$ docker image ls
REPOSITORY           TAG    IMAGE ID      CREATED         SIZE
flink-on-kubernetes  0.0.1  505d2f11cc57  10 seconds ago  618MB
```

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
* https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/
* https://jobs.zalando.com/tech/blog/running-apache-flink-on-kubernetes/
* https://www.slideshare.net/tillrohrmann/redesigning-apache-flinks-distributed-architecture-flink-forward-2017

[1]: https://github.com/kubernetes/minikube
[2]: https://www.virtualbox.org
[3]: https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
[4]: https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/darwin/amd64/kubectl
[5]: https://minikube.sigs.k8s.io/docs/reference/networking/proxy/
[6]: https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/datastream_api.html#example-program
[7]: https://github.com/jizhang/flink-on-kubernetes
[8]: https://kubernetes.io/
[9]: https://flink.apache.org/
[10]: https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=65147077
[11]: https://hub.docker.com/_/flink
[12]: https://github.com/docker-flink/docker-flink/blob/master/1.8/scala_2.12-debian/Dockerfile
[13]: https://repo.maven.apache.org/maven2/org/apache/flink/flink-shaded-hadoop-2-uber/2.8.3-7.0/flink-shaded-hadoop-2-uber-2.8.3-7.0.jar
