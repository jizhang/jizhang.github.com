---
title: Monitor Kubernetes Volume Storage
tags:
  - kubernetes
  - prometheus
  - devops
categories: Programming
date: 2022-07-01 08:21:39
---


Pods running on Kubernetes may claim a Persistent Volume to store data that last between pod restarts. This volume is usually of limited size, so we need to monitor its storage and alert for low free space. For stateless pods, it is also necessary to monitor its disk usage, since the application within may write logs or other contents directly onto the Docker writable layer. In Kubernetes terms, this space is called ephemeral storage. Another way to prevent ephemeral storge from filling up is to monitor the nodes' disk space directly. This article will demonstrate how to monitor volume storage with Prometheus.

## Monitor Persistent Volume

`kubelet` exposes the following metrics for Persistent Volumes:

```
$ curl http://10.0.0.1:10255/metrics
# HELP kubelet_volume_stats_capacity_bytes [ALPHA] Capacity in bytes of the volume
# TYPE kubelet_volume_stats_capacity_bytes gauge
kubelet_volume_stats_capacity_bytes{namespace="airflow",persistentvolumeclaim="data-airflow2-postgresql-0"} 4.214145024e+10
kubelet_volume_stats_capacity_bytes{namespace="default",persistentvolumeclaim="grafana"} 2.1003583488e+10

# HELP kubelet_volume_stats_used_bytes [ALPHA] Number of used bytes in the volume
# TYPE kubelet_volume_stats_used_bytes gauge
kubelet_volume_stats_used_bytes{namespace="airflow",persistentvolumeclaim="data-airflow2-postgresql-0"} 4.086779904e+09
kubelet_volume_stats_used_bytes{namespace="default",persistentvolumeclaim="grafana"} 4.9381376e+07
```

After you setup the [Prometheus Stack][1] with [Helm chart][2], you will get a Service and ServiceMonitor that help scraping these metrics. Then they can be queried in Prometheus UI:

![Prometheus UI](/images/k8s-volume/prometheus-ui.png)

<!-- more -->

And visualized with Grafana:

![Grafana](/images/k8s-volume/grafana.png)

Here is a simple alert rule that warns on disk usage:

```yaml
- alert: PrometheusPV
  expr: >
    sum(kubelet_volume_stats_used_bytes{persistentvolumeclaim="prometheus-prometheus-kube-prometheus-prometheus-db-prometheus-prometheus-kube-prometheus-prometheus-0"})
    / sum(kubelet_volume_stats_capacity_bytes{persistentvolumeclaim="prometheus-prometheus-kube-prometheus-prometheus-db-prometheus-prometheus-kube-prometheus-prometheus-0"})
    > 0.8
  labels:
    severity: critical
  annotations:
    description: Prometheus PV disk usage is greater than 80%.
```

## Monitor Ephemeral Storage

According to [Kubernetes documentation][3], ephemeral storage consists of `emptyDir`, logs, and the above-mentioned [writable container layer][7]. One can limit the use of ephemeral storage by configuring `resources` in container spec:

```yaml
kind: Pod
spec:
  containers:
  - name: app
    resources:
      requests:
        ephemeral-storage: "2Gi"
      limits:
        ephemeral-storage: "4Gi"
    volumeMounts:
    - name: ephemeral
      mountPath: "/tmp"
  volumes:
    - name: ephemeral
      emptyDir: {}
```

`kubelet` integrates the [cAdvisor][4] (Container Advisor) utility, which exposes a series of container metrics:

| Metric name | Type | Description | Unit |
| --- | --- | --- | --- |
| **container_fs_usage_bytes** | Gauge | Number of bytes that are consumed by the container on this filesystem | bytes |
| container_memory_working_set_bytes | Gauge | Current working set | bytes |
| container_cpu_usage_seconds_total | Counter | Cumulative cpu time consumed | seconds |
| container_network_transmit_bytes_total | Counter | Cumulative count of bytes transmitted | bytes |

To get the `limits` we specified in pod spec, we need the help of [`kube-state-metrics`][5] that exposes a metric named `kube_pod_container_resource_limits`:

| Metric name | Value |
| --- | --- |
| kube_pod_container_resource_limits{exported_pod="nginx-57bf55c5b5-n7vzp", resource="memory", unit="byte"} | 67108864 |
| kube_pod_container_resource_limits{exported_pod="nginx-57bf55c5b5-n7vzp", resource="cpu", unit="core"} | 0.1 |
| kube_pod_container_resource_limits{exported_pod="nginx-57bf55c5b5-n7vzp", resource="ephemeral_storage", unit="byte"} | 1073741824 |

If a pod is using more disk space than expected, it is usually because of application logs. One can adjust the log level, mount a dedicated PV for logging, or clear log files periodically. To temporarily solve the alert, just restart the Deployment or StatefulSet.

## Monitor Node Disk Space

Though all cloud infrastructure providers have out-of-the-box warnings for virtual machines' disk space, we can still setup our own graphs and alerts. Prometheus has built-in [`node-exporter`][6] metrics:

```
sum(
    max by (device) (
        node_filesystem_size_bytes{job="node-exporter", instance="10.0.0.1:9100", fstype!=""}
    -
        node_filesystem_avail_bytes{job="node-exporter", instance="10.0.0.1:9100", fstype!=""}
    )
)
```

![Node filesystem size](/images/k8s-volume/node-filesystem-size.png)

[1]: https://github.com/prometheus-operator/kube-prometheus
[2]: https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack
[3]: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-emphemeralstorage-consumption
[4]: https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md
[5]: https://github.com/kubernetes/kube-state-metrics/tree/master/docs
[6]: https://github.com/prometheus/node_exporter
[7]: https://docs.docker.com/storage/storagedriver/
