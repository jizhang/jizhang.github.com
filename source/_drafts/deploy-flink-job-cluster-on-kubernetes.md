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
    .socketTextStream("192.168.99.1", 9999)
    .flatMap(new Splitter())
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1);

dataStream.print();
```

IP `192.168.99.1` allows container to access services running on minikube host. For this example to work, you need to run `nc -lk 9999` on your host before creating the JobManager pod.

Run `mvn clean package`, and the compiled job jar can be found in `target/flink-on-kubernetes-0.0.1-SNAPSHOT-jar-with-dependencies.jar`.

## Build Docker Image

Flink provides an official docker image on [DockerHub][11]. We can use it as the base image and add job jar into it. Besides, in recent Flink distribution, the Hadoop binary is not included anymore, so we need to add Hadoop jar as well. Take a quick look of the base image's [Dockerfile][12], it does the following tasks:

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

Before building the image, you need to install Docker CLI and point it to the docker service inside minikube:

```bash
$ brew install docker
$ eval $(minikube docker-env)
```

Then, download the [Hadoop uber jar][12], and execute the following commands:

```bash
$ cd /path/to/Dockerfile
$ cp /path/to/flink-shaded-hadoop-2-uber-2.8.3-7.0.jar hadoop.jar
$ cp /path/to/flink-on-kubernetes-0.0.1-SNAPSHOT-jar-with-dependencies.jar job.jar
$ docker build --build-arg hadoop_jar=hadoop.jar --build-arg job_jar=job.jar --tag flink-on-kubernetes:0.0.1 .
```

Now we have a local docker image that is ready to be deployed.

```bash
$ docker image ls
REPOSITORY           TAG    IMAGE ID      CREATED         SIZE
flink-on-kubernetes  0.0.1  505d2f11cc57  10 seconds ago  618MB
```

## Deploy JobManager

First, we create a k8s Job for Flink JobManager. Job and Deployment both create and manage Pods to do some work. The difference is Job will quit if the Pod finishes successfully, based on the exit code, while Deployment only quits when asked to. This feature enables us to cancel the Flink job manually, without worrying Deployment restarts the JobManager by mistake.

Here's the `jobmanager.yml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ${JOB}-jobmanager
spec:
  template:
    metadata:
      labels:
        app: flink
        instance: ${JOB}-jobmanager
    spec:
      restartPolicy: OnFailure
      containers:
      - name: jobmanager
        image: flink-on-kubernetes:0.0.1
        command: ["/opt/flink/bin/standalone-job.sh"]
        args: ["start-foreground",
               "-Djobmanager.rpc.address=${JOB}-jobmanager",
               "-Dparallelism.default=1",
               "-Dblob.server.port=6124",
               "-Dqueryable-state.server.ports=6125"]
        ports:
        - containerPort: 6123
          name: rpc
        - containerPort: 6124
          name: blob
        - containerPort: 6125
          name: query
        - containerPort: 8081
          name: ui
```

* `${JOB}` can be replaced by `envsubst`, so that config files can be reused by different jobs.
* Container's entry point is changed to `standalone-job.sh`. It will start the JobManager in foreground, scan the class path for a `Main-Class`, or you can specify the full class name via `-j` option.
* JobManager's RPC address is the k8s [Service][14]'s name, which we will create later. Other containers can access JobManager via this host name.
* Blob server and queryable state server's ports are by default random. We change them to fixed ports for easy exposure.

```bash
$ export JOB=flink-on-kubernetes
$ envsubst <jobmanager.yml | kubectl create -f -
$ kubectl get pod
NAME                                   READY   STATUS    RESTARTS   AGE
flink-on-kubernetes-jobmanager-kc4kq   1/1     Running   0          2m26s
```

Next, we expose this JobManager as k8s Service, so that TaskManagers can register to it.

`service.yml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ${JOB}-jobmanager
spec:
  selector:
    app: flink
    instance: ${JOB}-jobmanager
  type: NodePort
  ports:
  - name: rpc
    port: 6123
  - name: blob
    port: 6124
  - name: query
    port: 6125
  - name: ui
    port: 8081
```

`type: NodePort` is necessary because we also want to interact with this JobManager outside the k8s cluster.

```bash
$ envsubst <service.yml | kubectl create -f -
$ kubectl get service
NAME                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                      AGE
flink-on-kubernetes-jobmanager   NodePort    10.109.78.143   <none>        6123:31476/TCP,6124:32268/TCP,6125:31602/TCP,8081:31254/TCP  15m
```

We can see Flink dashboard is exposed on port 31254 on the virtual machine. Minikube provides a command to retrieve the full url of a service.

```bash
$ minikube service $JOB-jobmanager --url
http://192.168.99.108:31476
http://192.168.99.108:32268
http://192.168.99.108:31602
http://192.168.99.108:31254
```

## Deploy TaskManager

`taskmanager.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${JOB}-taskmanager
spec:
  selector:
    matchLabels:
      app: flink
      instance: ${JOB}-taskmanager
  replicas: 1
  template:
    metadata:
      labels:
        app: flink
        instance: ${JOB}-taskmanager
    spec:
      containers:
      - name: taskmanager
        image: flink-on-kubernetes:0.0.1
        command: ["/opt/flink/bin/taskmanager.sh"]
        args: ["start-foreground", "-Djobmanager.rpc.address=${JOB}-jobmanager"]
```

Change the number of `replicas` to add more TaskManagers. The `taskmanager.numberOfTaskSlots` is set to `1` in this image, which is recommended because we should let k8s handle the scaling.

Now the job cluster is running, try typing something into the `nc` console:

```bash
$ nc -lk 9999
hello world
hello flink
```

Open another terminal and tail the TaskManager's standard output:

```bash
$ kubectl logs -f -l instance=$JOB-taskmanager
(hello,2)
(flink,1)
(world,1)
```

## Configure JobManager HA

## Manage Flink Job

### Stop and Resume

### Scale

## Logs and Monitoring

https://stackoverflow.com/a/55871120/1030720

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
[14]: https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies
