---
title: Monitor Kubernetes Volume Storage
tags: [kubernetes, prometheus]
categories: DevOps
---

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
