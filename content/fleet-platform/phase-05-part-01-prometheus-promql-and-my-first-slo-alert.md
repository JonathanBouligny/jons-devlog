---
title: "Phase 5, Part 1: Prometheus, PromQL and My First SLO Alert"
date: 2026-10-28
series: ["fleet-platform"]
tags: ["prometheus", "promql", "grafana", "slo", "blackbox-exporter", "kubernetes"]
phase: "5"
summary: "Installing kube-prometheus-stack, learning PromQL from the `up` metric through histograms, then probing Grafana with blackbox exporter and wiring a multi-window burn-rate SLO alert from the Google SRE Workbook."
draft: false
---

The full platform is on GitHub: [github.com/JonathanBouligny/fleet-platform](https://github.com/JonathanBouligny/fleet-platform). This post covers the monitoring stack and the first SLO alert.

So I installed kube-prometheus-stack, which includes Alertmanager, Prometheus, Grafana and everything else the chart comes with.

```text
$ kubectl get pods,pvc,svc -n monitoring
NAME                                                         READY   STATUS    RESTARTS   AGE
pod/alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0          42h
pod/prometheus-grafana-6c5d845bc5-2bhmw                      3/3     Running   0          18h
pod/prometheus-kube-prometheus-operator-6c6dbff88-5b4bh      1/1     Running   0          42h
pod/prometheus-kube-state-metrics-6bfc9c7c48-79zr7           1/1     Running   0          42h
pod/prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0          42h
pod/prometheus-prometheus-node-exporter-mxgkb                1/1     Running   0          42h
pod/prometheus-prometheus-node-exporter-p8dmv                1/1     Running   0          42h

NAME                                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-operated                     ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   42h
service/prometheus-grafana                        ClusterIP   10.43.164.175   <none>        80/TCP                       42h
service/prometheus-kube-prometheus-alertmanager   ClusterIP   10.43.143.112   <none>        9093/TCP,8080/TCP            42h
service/prometheus-kube-prometheus-operator       ClusterIP   10.43.241.200   <none>        443/TCP                      42h
service/prometheus-kube-prometheus-prometheus     ClusterIP   10.43.246.45    <none>        9090/TCP,8080/TCP            42h
service/prometheus-kube-state-metrics             ClusterIP   10.43.18.188    <none>        8080/TCP                     42h
service/prometheus-operated                       ClusterIP   None            <none>        9090/TCP                     42h
service/prometheus-prometheus-node-exporter       ClusterIP   10.43.110.128   <none>        9100/TCP                     42h
```

## Getting the Grafana password in

The next thing is getting the values file together, specifically for a password. That was as easy as finding the value in the values.yml file.

```yaml
grafana:
  admin:
    ## Name of the secret. Can be templated.
    existingSecret: "grafana-admin"
    userKey: admin-user
    passwordKey: admin-password
```

Then creating a secret matching that value and adding it to the secret preloader.

```yaml
---
- name: Configure bootstrap secrets
  hosts: localhost
  vars:
    kubeconfig_path: "~/.kube/config"
    secrets_dir: "~/.secrets/fleet-platform" # Stored locally outside the repo
    bootstrap_secrets:
      - { namespace: argocd, file: "{{ secrets_dir }}/secret-argo-forgejo.yml" }
      - { namespace: cert-manager, file: "{{ secrets_dir }}/cloudflare-api-token.yml" }
      - { namespace: haproxy-ingress-internal, file: "{{ secrets_dir }}/wehaveaproblem-net.yml" }
      - { namespace: monitoring, file: "{{ secrets_dir }}/grafana-admin.yml" }
  tasks:
    - name: Make sure namespaces for the secrets exist
      kubernetes.core.k8s:
        kubeconfig: "{{ kubeconfig_path }}"
        state: present
        name: "{{ item.namespace }}"
        kind: Namespace
      loop: "{{ bootstrap_secrets }}"

    - name: Apply secrets in loop
      kubernetes.core.k8s:
        kubeconfig: "{{ kubeconfig_path }}"
        namespace: "{{ item.namespace }}"
        state: present
        apply: true
        src: "{{ item.file }}"
      loop: "{{ bootstrap_secrets }}"
      no_log: true
```

Alright! After that we added ingresses for Grafana, Alertmanager and Prometheus so we can get at all three GUIs.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
spec:
  ingressClassName: haproxy-internal
  rules:
    - host: "grafana.wehaveaproblem.net"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-grafana
                port:
                  number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: alert-manager
  namespace: monitoring
spec:
  ingressClassName: haproxy-internal
  rules:
    - host: "alertmanager.wehaveaproblem.net"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-kube-prometheus-alertmanager
                port:
                  number: 9093
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus
  namespace: monitoring
spec:
  ingressClassName: haproxy-internal
  rules:
    - host: "prometheus.wehaveaproblem.net"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-kube-prometheus-prometheus
                port:
                  number: 9090
```

## Messing with Prometheus

From there I started messing with Prometheus.

A little about Prometheus first. Prometheus is a monitoring suite that collects time series data about an endpoint. I got to work looking at the data Prometheus collects by default before creating any new data.

First we stuck with the tried and true `up` metric. In Prometheus an instance is any monitored endpoint, specifically could we scrape from it, and a job is a collection of endpoints associated with an object, say the kubelet.

```promql
up
```

| job | instance | metrics_path / container | value |
|---|---|---|---|
| kubelet | 192.168.1.20:10250 | /metrics | 1 |
| kubelet | 192.168.1.20:10250 | /metrics/cadvisor | 1 |
| kubelet | 192.168.1.20:10250 | /metrics/probes | 1 |
| kubelet | 192.168.1.21:10250 | /metrics | 1 |
| kubelet | 192.168.1.21:10250 | /metrics/cadvisor | 1 |
| kubelet | 192.168.1.21:10250 | /metrics/probes | 1 |
| apiserver | 192.168.1.20:6443 | | 1 |
| node-exporter | 192.168.1.20:9100 | node-exporter | 1 |
| node-exporter | 192.168.1.21:9100 | node-exporter | 1 |
| prometheus-kube-prometheus-prometheus | 10.42.1.15:9090 | prometheus | 1 |
| prometheus-kube-prometheus-prometheus | 10.42.1.15:8080 | config-reloader | 1 |
| prometheus-kube-prometheus-alertmanager | 10.42.1.14:9093 | alertmanager | 1 |
| prometheus-kube-prometheus-alertmanager | 10.42.1.14:8080 | config-reloader | 1 |
| prometheus-kube-prometheus-operator | 10.42.1.13:10250 | kube-prometheus-stack | 1 |
| kube-state-metrics | 10.42.0.12:8080 | kube-state-metrics | 1 |
| coredns | 10.42.0.4:9153 | coredns | 1 |
| prometheus-grafana | 10.42.0.20:3000 | grafana | 1 |

17 series. Every row also carries `namespace`, `pod`, `service` and `endpoint` labels, trimmed here.

This list from running `up` just tells us which endpoints we're able to scrape from or not.

So we can use `count by (job)` to group the instances by job.

```promql
count by (job) (up)
```

| job | count |
|---|---|
| kubelet | 6 |
| apiserver | 1 |
| prometheus-kube-prometheus-alertmanager | 2 |
| prometheus-kube-prometheus-prometheus | 2 |
| node-exporter | 2 |
| kube-state-metrics | 1 |
| coredns | 1 |
| prometheus-kube-prometheus-operator | 1 |
| prometheus-grafana | 1 |

This says there are 6 instances associated with the kubelet job, and if you look at the `up` table above there are 3 endpoints per node and 2 nodes.

Next is a more useful metric, `node_cpu_seconds_total`. Node exporter can be scraped for mode and time series data into `node_cpu_seconds_total`. On its own the data is a little useless.

```promql
node_cpu_seconds_total
```

| instance | cpu | mode | value (seconds) |
|---|---|---|---|
| 192.168.1.20:9100 | 0 | idle | 218658.53 |
| 192.168.1.20:9100 | 0 | iowait | 154.16 |
| 192.168.1.20:9100 | 0 | irq | 0 |
| 192.168.1.20:9100 | 0 | nice | 1.84 |
| 192.168.1.20:9100 | 0 | softirq | 518.35 |
| 192.168.1.20:9100 | 0 | steal | 15.52 |
| 192.168.1.20:9100 | 0 | system | 1080.14 |
| 192.168.1.20:9100 | 0 | user | 3230.68 |
| 192.168.1.20:9100 | 1 | idle | 218684.56 |
| 192.168.1.20:9100 | 1 | iowait | 160.87 |
| ... | ... | ... | ... |

64 series: 8 modes x 4 cpus x 2 nodes.

But if we add the `rate` function we can get some very useful information. `rate` calculates how fast whatever it is counting climbs per second. In this case it outputs a fraction and tells us how much of a 5 minute window the mode is taking up.

```promql
rate(node_cpu_seconds_total[5m])
```

| instance | cpu | mode | value (per second) |
|---|---|---|---|
| 192.168.1.20:9100 | 0 | idle | 0.9756 |
| 192.168.1.20:9100 | 0 | iowait | 0.0009 |
| 192.168.1.20:9100 | 0 | irq | 0 |
| ... | ... | ... | ... |

64 series again, one per cpu and mode. The idle row says cpu 0 spent 97.6% of the last 5 minutes idle.

Now if we add a selection for a specific mode like idle we can see how much time each core spends in idle mode on each node.

```promql
rate(node_cpu_seconds_total{mode="idle"}[5m])
```

| instance | cpu | value |
|---|---|---|
| 192.168.1.20:9100 | 0 | 0.9738 |
| 192.168.1.20:9100 | 1 | 0.9730 |
| 192.168.1.20:9100 | 2 | 0.9731 |
| 192.168.1.20:9100 | 3 | 0.9718 |
| 192.168.1.21:9100 | 0 | 0.9826 |
| 192.168.1.21:9100 | 1 | 0.9840 |
| 192.168.1.21:9100 | 2 | 0.9829 |
| 192.168.1.21:9100 | 3 | 0.9834 |

So the full expression averages the cores by instance. It averages the rate the cpu spent in idle per second.

```promql
avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
```

| instance | value |
|---|---|
| 192.168.1.20:9100 | 0.9739 |
| 192.168.1.21:9100 | 0.9835 |

Then if we calculate this we actually get average cpu usage over a 5 minute window, the percentage of time spent outside of idle mode!

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))) * 100
```

| instance | cpu used (%) |
|---|---|
| 192.168.1.20:9100 | 2.66 |
| 192.168.1.21:9100 | 1.67 |

Next we did percentage of unused memory. Memory is a gauge, so there is no `rate` here, just a division.

```promql
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
```

| instance | available (fraction) |
|---|---|
| 192.168.1.20:9100 | 0.6516 |
| 192.168.1.21:9100 | 0.8102 |

Then memory used, which is just 1 minus that.

```promql
1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
```

| instance | used (fraction) |
|---|---|
| 192.168.1.20:9100 | 0.3584 |
| 192.168.1.21:9100 | 0.2034 |

This is a subtle one. I did bytes available on a file system divided by total bytes on a file system and filtered out the tmpfs file system types.

```promql
node_filesystem_avail_bytes / node_filesystem_size_bytes{fstype!="tmpfs"}
```

The thing is it worked. Something subtle happens here. When you divide one metric by another, PromQL pairs up each of the values by matching identical labels. Since I excluded tmpfs from one side it excluded it from the other, because it only tried to divide the pairs. The explicit version is this, which just makes it clear.

```promql
node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"}
```

Next we learned about `increase`. The explanation is weird but what it actually means is it calculates how much a given value went up in a given timeframe. So when we run this it literally gives us the number of container restarts in the last hour.

```promql
increase(kube_pod_container_status_restarts_total[1h])
```

| namespace | pod | container | restarts in 1h |
|---|---|---|---|
| monitoring | prometheus-prometheus-node-exporter-mxgkb | node-exporter | 0 |
| argocd | argocd-redis-c4fdd947-4ml4t | redis | 0 |
| cert-manager | cert-manager-webhook-558c6d4f4d-htrjf | cert-manager-webhook | 0 |
| monitoring | prometheus-kube-prometheus-operator-6c6dbff88-5b4bh | kube-prometheus-stack | 0 |
| argocd | argocd-repo-server-7fd96b6d87-k7rhl | repo-server | 0 |
| default | guestbook-ui-5d6468fd55-57nbp | guestbook-ui | 0 |
| kube-system | coredns-54996dc9b4-d86db | coredns | 0 |
| argocd | argocd-application-controller-0 | application-controller | 0 |
| ... | ... | ... | ... |

37 series, all 0. Nothing has restarted in the last hour.

The next one is memory per pod. This gives us the total amount of memory used at an instant of time per pod. We do this by summing the memory of all the containers by pod. `sum` always grabs an instant of time, it is not summing over time.

```promql
sum by (pod) (container_memory_working_set_bytes)
```

| pod | bytes | MiB |
|---|---|---|
| prometheus-prometheus-kube-prometheus-prometheus-0 | 1033609216 | 986 |
| argocd-application-controller-0 | 685572096 | 654 |
| cert-manager-cainjector-6468bc96c7-bbwg4 | 87932928 | 84 |
| argocd-repo-server-7fd96b6d87-k7rhl | 84955136 | 81 |
| alertmanager-prometheus-kube-prometheus-alertmanager-0 | 70520832 | 67 |
| argocd-server-7647b49864-nq5nc | 57892864 | 55 |
| cert-manager-76ffbfcbfc-2tj5w | 51552256 | 49 |
| prometheus-kube-prometheus-operator-6c6dbff88-5b4bh | 50356224 | 48 |
| argocd-applicationset-controller-7c7f547b68-5gmk2 | 46415872 | 44 |
| argocd-dex-server-7d7dcff6f4-ssrv5 | 44691456 | 43 |
| argocd-notifications-controller-fd7558bd5-d92v2 | 42352640 | 40 |
| cert-manager-webhook-558c6d4f4d-htrjf | 27357184 | 26 |
| prometheus-prometheus-node-exporter-p8dmv | 22110208 | 21 |
| prometheus-prometheus-node-exporter-mxgkb | 22102016 | 21 |
| argocd-redis-c4fdd947-4ml4t | 21618688 | 21 |
| guestbook-ui-5d6468fd55-57nbp | 18366464 | 18 |
| local-path-provisioner-77b9867795-b44nc | 16564224 | 16 |

25 series, sorted by size here. Prometheus itself is the biggest thing on the cluster at almost 1 GiB.

### What I learned

I learned that an instance represents an endpoint and a job represents a collection of instances. Each line in Prometheus reflects a combination of job and instance, it's a unique row. Each row represents a set of time series data. There are three types of data, counters which can only go up or be reset to zero, gauges which can go up and down arbitrarily, and histograms which are buckets of data.

We started by looking at the Prometheus GUI and looking at the data set for `up`. `up` is the basic one, it checks to see if monitored endpoints have been scraped or not. Then we can check things like `node_cpu_seconds_total`, which tells us how long the cpu spent in a specific mode. `node_cpu_seconds_total` is a counter because you can't take back time, it can only go up. Then we can use that to calculate used and free cpu percentages. We learned to use `rate`, which only takes counters like `node_cpu_seconds_total`. Rate is literally the rate per second of something increasing. We can do the same with memory, which is a gauge. We can calculate the available memory percentage and then calculate used from there. We can look at available storage for file systems and we learned how to use filters, which allowed us to filter out the tmpfs file systems. We learned how to use the `increase` function, which takes in a set of time filtered data, for example 1h for 1 hour, and shows how much the number has increased over that time period. We learned to check which pods are not running by also using filters. We learned to sum gauges with the aggregate function by label syntax, in this case `sum by (pod)`, and we used that to calculate the memory used by all the containers in a pod.

Then we learned about histograms, which use buckets to group metrics, and this is where we combined everything together. First we got the rate of change per second over 5 minutes for each instance, then we summed all the rates that match up with the different buckets using the `le` label, which is the bucket boundary, then we grabbed the 95th percentile value. The metric was `prometheus_http_request_duration_seconds_bucket`, so the whole thing told us how slow the slowest 5% of requests were.

## Alright, time for SLIs and SLOs

We're going to be measuring the uptime of grafana.wehaveaproblem.net (which is my internal domain, not externally reachable). To do that we need blackbox exporter, a monitoring system that mimics what it's like to access the cluster from the outside, from inside the cluster. We install blackbox with helm, specifically we make an Argo CD application, that's easy enough. Then we set up a `Probe`, which is a Prometheus Operator CRD that defines how to scrape metrics using an external exporter.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Probe
metadata:
  name: grafana-blackbox
  namespace: monitoring
  labels:
    release: prometheus
spec:
  interval: 30s
  module: http_2xx
  prober:
    url: blackbox-prometheus-blackbox-exporter:9115   # match the service name (host:port, no https://)
  targets:
    staticConfig:
      static:
        - https://grafana.wehaveaproblem.net
```

I defined the exporter in a folder in my app of apps folder so that later when the cluster is rebuilt it will be rebuilt with the rest of the pieces.

We go to Prometheus and...

```promql
probe_success{instance="https://grafana.wehaveaproblem.net"}
```

| instance | job | value |
|---|---|---|
| https://grafana.wehaveaproblem.net | probe/monitoring/grafana-blackbox | 1 |

SUCCESS!

## Next we need to configure an alert

We configure an alert using a `PrometheusRule`, a CRD that defines alerting and recording rules to be evaluated by Prometheus. Recording rules are precomputed metrics computed on an interval, and alerting rules are rules that... alert.

So we create this PrometheusRule file along with the Google SRE Workbook.

First we need to state our SLI and SLO.

Our SLI is: the proportion of HTTP probes to https://grafana.wehaveaproblem.net that return 2xx within 5 seconds, measured from inside the cluster every 30 seconds.

Our SLO is: 99% of probes succeed, measured over a rolling 30 days.

By default blackbox has a 5 second timeout for HTTP requests, that's where the 5 seconds comes from, and the 30 seconds is the interval in our Probe. The 30 days is the standard SLO window from the SRE Workbook, and it's baked into the math below. (Prometheus only keeps 10 days of data here. That's the kube-prometheus-stack chart default, a standard Prometheus install keeps 15. It doesn't matter for the alerts because the rules only ever look back 6 hours.)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: grafana-slo
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: grafana-slo.rules
      rules:
        - record: probe:grafana:error_ratio5m
          expr: 1 - avg_over_time(probe_success{instance="https://grafana.wehaveaproblem.net"}[5m])
        - record: probe:grafana:error_ratio1h
          expr: 1 - avg_over_time(probe_success{instance="https://grafana.wehaveaproblem.net"}[1h])
        - record: probe:grafana:error_ratio30m
          expr: 1 - avg_over_time(probe_success{instance="https://grafana.wehaveaproblem.net"}[30m])
        - record: probe:grafana:error_ratio6h
          expr: 1 - avg_over_time(probe_success{instance="https://grafana.wehaveaproblem.net"}[6h])
        - alert: GrafanaSLOFastBurn
          expr: probe:grafana:error_ratio1h > 0.144 and probe:grafana:error_ratio5m > 0.144
          for: 2m
          labels:
            severity: page
          annotations:
            summary: "Grafana SLO fast burn (>14.4x budget)"
        - alert: GrafanaSLOSlowBurn
          expr: probe:grafana:error_ratio6h > 0.06 and probe:grafana:error_ratio30m > 0.06
          for: 15m
          labels:
            severity: ticket
          annotations:
            summary: "Grafana SLO slow burn (>6x budget)"
```

### Where the numbers come from

OK SO we use this file. This file and its values start from the standard Google SRE Workbook. We declare an SLO of 99% of probes to Grafana succeed over a rolling 30 days. A probe counts as successful if grafana.wehaveaproblem.net returns a 2xx response within 5 seconds. We chose the 30 days because it's standard and baked into the math in the manifest file. We are probing grafana.wehaveaproblem.net with the blackbox exporter and it is using a 5 second timeout for probing. That corresponds to the within 5 seconds. If blackbox exporter times out then the probe returns zero.

Now the math. We use the standard Google SRE Workbook targets of 2% of our error budget consumed in 1 hour and 5% of our error budget consumed in 6 hours. We need to translate this into actual alert thresholds, and in order to do that we turn the 2% in 1 hour into a burn rate. The burn rate equation is

```text
burn rate = (share of the error budget consumed * SLO window in hours) / (alert window in hours)
```

That turns into 0.02 (the 2%), 720 hours for 30 days, and the window Google declared, which is 1 hour. That's (0.02 * 720) / 1, which turns into 14.4. The numerator is how many months' worth of the budget you'd burn in one month (720 hours) if you lost 2% every single hour, where the 2% comes from the Google SRE Workbook and the 720 comes from our 30 day SLO. The denominator is the alert window. 720 divided by it is how many of those windows fit in a month, so dividing by 1 hour leaves it at 14.4, and the 6 hour alert divides by 6 instead, which gives (0.05 * 720) / 6 = 6.

We then take that burn rate and multiply it by the error budget, which is 0.01 or 1%, because we set our SLO to 99% uptime so 1% downtime. Prometheus only sees the fraction of the probes that failed, so we convert the burn rate into that fraction by multiplying it by the error budget. A burn rate of 14.4 means we are failing 14.4 times faster than allowed, and the allowed rate is 1% of probes, so 14.4 times 1% is 14.4% of probes failing, or 0.144. That is the number Prometheus can compare against. The slow burn alert is the same thing with 6, so 6 * 0.01 = 0.06.

Each alert has two windows because of what happens after a fix. If the fast burn alert only looked at the 1 hour window, it would keep firing for about 50 minutes after I fixed the problem, because the failed probes stay inside the window until they age out. The 5 minute window only stays true while failures are still happening. Both have to be true, so the alert clears about 4.5 minutes after the fix instead of 50. The `for` is a debounce on top of that. The condition has to hold for 2 minutes (or 15 for slow burn) before the alert goes from Pending to Firing.

## It actually happened

While I was writing this up I moved the HAProxy ingress controller to a new namespace. I wanted it in `haproxy-ingress-internal` instead of `haproxy-controller`, which meant changing `spec.destination.namespace` on the Argo CD Application, moving the cert-manager Certificate, and copying the TLS secret across. Argo CD pruned the old controller before the new one had bound ports 80 and 443 (k3s ServiceLB only lets one LoadBalancer service hold a host port), and for a few minutes nothing behind the ingress was reachable, including Grafana. The probe caught it.

For that hour I asked Prometheus how many probes it had and how many succeeded.

```promql
count_over_time(probe_success{instance="https://grafana.wehaveaproblem.net"}[1h])
```

| instance | value |
|---|---|
| https://grafana.wehaveaproblem.net | 120 |

```promql
avg_over_time(probe_success{instance="https://grafana.wehaveaproblem.net"}[1h])
```

| instance | value |
|---|---|
| https://grafana.wehaveaproblem.net | 0.65 |

120 probes is right for a 30 second interval over an hour. A success average of 0.65 is 78 successes and 42 failures, an error ratio of 0.35.

Running the numbers from above, the burn rate is 0.35 / 0.01 = 35, so we were burning budget 35 times faster than the SLO allows. Over one hour that is 35 * (1 / 720) = 4.9% of the whole month's error budget. In minutes, 42 failed probes * 30 seconds is about 21 minutes of downtime, out of the 432 minutes (1% of 30 days) the SLO allows. And 0.35 is well above the 0.144 threshold, so the fast burn alert would have paged.

Would have, because I hadn't committed the PrometheusRule yet when it happened. The rules file was still sitting untracked in my repo. So the first real incident on this platform is also the reason the alerts are live now.
