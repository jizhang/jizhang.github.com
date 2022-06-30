---
title: Monitor Kubernetes Volume Storage
tags: [kubernetes, prometheus]
categories: DevOps
---

Pods running on Kubernetes may claim a Persistent Volume to store data that last between pod restarts. This volume is usually of limited size, so we need to monitor its storage and alert for high usage. For stateless pods, it is also necessary to monitor its disk usage, since the application within may write logs or other contents directly onto the Docker writable layer. In Kubernetes terms, this space is called ephemeral storage. Another way to prevent ephemeral storge from filling up is to monitor the nodes' disk space directly. This article will demonstrate how to monitor volume storage with Prometheus.

## Monitor Persistent Volume

`kubelet` exposes the following metrics for Persistem Volumes:

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

After you setup the [Prometheus Stack][1] with [Helm chart][2], you will get a Service and ServiceMonitor that help scraping these metrics. Then they can be queried by Prometheus UI:

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

## Monitor Node Disk Space


[1]: https://github.com/prometheus-operator/kube-prometheus
[2]: https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack


prometheus-operator-prometheus-node-exporter
prometheus-kubelet
cadvisor
metrics-server vs kube-state-metrics

Node
    sum(
        max by (device) (
            node_filesystem_size_bytes{job="node-exporter", instance="10.111.8.12:9100", fstype!=""}
        -
            node_filesystem_avail_bytes{job="node-exporter", instance="10.111.8.12:9100", fstype!=""}
        )
    )

PV
    sum(kubelet_volume_stats_used_bytes{persistentvolumeclaim="prometheus-prometheus-kube-prometheus-prometheus-db-prometheus-prometheus-kube-prometheus-prometheus-0"})
        / sum(kubelet_volume_stats_capacity_bytes{persistentvolumeclaim="prometheus-prometheus-kube-prometheus-prometheus-db-prometheus-prometheus-kube-prometheus-prometheus-0"})
        > 0.8
    kube_persistentvolume_capacity_bytes?

Ephemeral
    container_fs_usage_bytes{pod!=""} > 5000000000


https://kubernetes.io/docs/concepts/storage/volumes/
https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#local-ephemeral-storage
    emptyDir volumes, except tmpfs emptyDir volumes
    directories holding node-level logs
    writeable container layers
https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md

k8s 启动一个容器的时候，如果没有挂任何的 pv，那容器在往文件系统里写东西的时候其实是写在一个叫 writable layer 的地方，也就是宿主机的磁盘上。如果不加限制，可能会把宿主机磁盘写满。限制的方式是用 resources.limits.ephemeral-storage 来指定能用多少空间，并且可以用 container_fs_usage_bytes 和 kube_pod_container_resource_limits 来监控使用率？
