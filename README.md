# LVM Exporter Helm Chart

This chart deploys Prometheus `lvm-exporter` as a DaemonSet on each Linux node so each host exposes its local LVM status through Prometheus.

## Why this shape

- `DaemonSet` ensures one exporter instance per node.
- The chart mounts the host LVM state from `/run/lvm`, `/etc/lvm`, `/var/lib/lvm`, and `/dev`.
- It is designed for RKE2 on RHEL 9 where SELinux is enabled and the host LVM metadata is available at the host paths above.
- A `CiliumNetworkPolicy` is enabled by default for a Cilium-first environment, with the standard Kubernetes `NetworkPolicy` kept available as an optional fallback.

## Install

```bash
helm install lvm-exporter ./charts/lvm-exporter \
  --namespace monitoring \
  --create-namespace
```

The default values are already aligned with the monitoring namespace and the `kube-prometheus-stack-1` release name.

## RHEL 9 / SELinux notes

The exporter runs with a constrained container security context but still requires host-level access to the LVM sockets and metadata. If your environment enforces stricter SELinux policies than the defaults, you may need to:

- mount the host LVM directories exactly as shown in the chart,
- ensure the host policy allows the container to read `/run/lvm` and `/etc/lvm`,
- review the host Cilium policy for the `lvm-exporter` pods,
- or temporarily set `securityContext.privileged: true` only if your cluster hardening requires it.

## Values of interest

```yaml
image:
  repository: mwennrich/lvm-exporter
  tag: "0.6.0"
  pullPolicy: IfNotPresent

serviceMonitor:
  enabled: true
  namespace: monitoring
  labels:
    release: kube-prometheus-stack-1

networkPolicy:
  enabled: true
  prometheusNamespace: kube-prometheous-stack-1
  prometheusPodSelector:
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/component: prometheus
```

## Metrics endpoint

The exporter exposes metrics on:

- `http://<node>:9080/metrics`

This is scraped by the kube-prometheus-stack `ServiceMonitor` when the labels match the Prometheus release.
