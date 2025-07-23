---
layout: post
title:  "Victoriametrics k8s stack setup gotchas"
---

## Use case

We want to **monitor** our Kubernetes cluster using [VictoriaMetrics](https://victoriametrics.com/) stack using the **single version**. The stack includes VictoriaMetrics Operator, Grafana dashboards, ServiceScrapes and VMRules, for a full overview of the stack see the [official documentation](https://docs.victoriametrics.com/helm/victoriametrics-k8s-stack/)

Before diving into the setup I want to point out the differences between **monitoring** and **observability**. Monitoring is about collecting metrics and logs from the system, while observability is about understanding the system's behavior and performance at any given state, **even unknowns states**[^1]. While the former is mainly reactive and focused on alerting for known/past issues, the latter aim to provide insights to complex systems, e.g. distributed systems, allowing us to proactively identify root causes. In this case, we are focusing on monitoring, as a complementary solution to a more comprehensive observability solution.

Moreover I would like to stress that we have been using the stack in production and we are greatly satisfied with it, the stack is easy to set up and maintain, and it provides a good overview of the cluster's health and performance while being cost-effective.

## Scraping is done explicitly

Whether you are migrating from Prometheus or using other components that rely on prometheus annotations for auto-discovery  (such as prometheus.io/port, prometheus.io/scrape, prometheus.io/path, etc.), you will need to explicitly define the `VMPodScrape` or `VMServiceScrape` resources for each service/workload you want to scrape. The [official documentation](https://docs.victoriametrics.com/operator/integrations/prometheus/#auto-discovery-for-prometheusio-annotations) provides examples on how to mimic/migrate the auto-discovery annotations by creating the corresponding `VMPodScrape` or `VMServiceScrape` resources.

## Disable control-plane metrics collection and alerting for cloud managed clusters

When using cloud-managed Kubernetes clusters (such as GKE, EKS, AKS), you may want to disable the collection of control-plane metrics and alerting to avoid unnecessary noise in your monitoring setup. This can be achieved by modifying the [default helm values](https://docs.victoriametrics.com/helm/victoriametrics-k8s-stack/#parameters) for the `victoriametrics-k8s-stack` chart. This will prevent the operator from collecting control-plane metrics and generating alerts for them.

```yaml
kubeControllerManager:
  enabled: false
kubeEtcd:
  enabled: false
kubeProxy:
  enabled: false
kubeScheduler:
  enabled: false
kubeApiServer:
  enabled: false
```

## Setup retention period and storage, cpu and memory resources

By default, the `victoriametrics-k8s-stack` chart sets a retention period of 1 month for the metrics collected by VictoriaMetrics together with a storage request of `20Gi`. Depending on your collection throughput and the number of metrics collected, you may want to adjust these values to fit your needs. In our case, we noticed that the persistence volume was filling up quickly, which causes the [remote storage to stop accepting new data](https://victoriametrics.com/blog/vmstorage-retention-merging-deduplication/#free-disk-space-watcher-read-only-mode). Similarly you will like to measure and adjust the CPU and memory resources. To modify the retention period and storage request, you can use the following values in your `values.yaml` file:

```yaml
vmsingle:
  spec:
    # default is 30 days
    retentionPeriod: "7d" # depends on your use case
    resources:
      limits:
        memory: 5Gi # depends on your use case
      requests:
        memory: 1Gi # depends on your use case
    storage:
      resources:
        requests:
          storage: 20Gi # depends on your use case
```

## Upgrade the stack regularly and watchout for CRDs changes

I think the stack is quite stable but the pace of development is quite fast, so I recommend to upgrade the stack regularly to benefit from the latest features and bug fixes. However, be aware that some changes may require you to update your custom resources (CRDs). To check for differences between the current CRDs and the ones used by the latest version of the stack, you can use the following [commands](https://docs.victoriametrics.com/helm/victoriametrics-k8s-stack/#upgrade-guide):

```bash
# 1. check the changes in CRD
$ helm show crds vm/victoria-metrics-k8s-stack --version [YOUR_CHART_VERSION] | kubectl diff -f -

# 2. apply the changes (update CRD)
$ helm show crds vm/victoria-metrics-k8s-stack --version [YOUR_CHART_VERSION] | kubectl apply -f - --server-side
```

## Configure VMAlertmanager using a secret

The `victoriametrics-k8s-stack` comes with a bunch of dashboards, recording and alerting rules, but to get the most out of it, you will need to configure the `VMAlertmanager` to send alerts to your preferred notification channels. The recommended way to do this is by creating a secret with the alerting configuration and then referencing it in the `values.yaml` file. The secret should contain the alerting configuration in the `alertmanager.yaml` format, which is quite similar to the format used by Prometheus Alertmanager. Here is an example of how to reference the secret in the `values.yaml` file:

```yaml
alertmanager:
  spec:
    configSecret: <name_of_your_secret>
```
Given the reactive nature of monitoring, you will likely will need to tweak the routing and inhibition rules to avoid alert fatigue and ensure that you are only notified of the most relevant alerts.

## Mind the relabelConfigs option for the vmScrape resource that will scrape kubelet metrics

By default the `victoriametrics-k8s-stack` chart will create a `VMScrape` resource to scrape the kubelet metrics, but will add a `relabelConfigs` option to map all node labels to the collected metrics. This is useful to have the node labels available in the metrics, but it can also lead to a large number of labels being added to the metrics, which can increase the cardinality of the metrics and lead to performance issues. If you are not interested in having all node labels in the metrics, you can disable this option by modifying `values.yaml` file:

```yaml
kubelet:
    vmScrape:
        spec:
             relabelConfigs:
                # Uncomment if you want to map all node labels to the collected metrics
                # - action: labelmap  
                #   regex: __meta_kubernetes_node_label_(.+)
                - sourceLabels: [__metrics_path__]
                targetLabel: metrics_path
                - targetLabel: job
                replacement: kubelet
```

## Conclusion

In this post, we have covered some of the gotchas and best practices when setting up the VictoriaMetrics k8s stack. The stack is easy to set up and maintain, and it provides a good overview of the cluster's health and performance while being cost-effective. However, it is important to keep in mind the differences between monitoring and observability, and to configure the stack according to your needs. Regularly upgrading the stack and watching out for CRDs changes is also recommended to benefit from the latest features and bug fixes.


[^1]: Honeycomb’s O’Reilly Book *Observability Engineering: Achieving Production Excellence*  
By Charity Majors, Liz Fong-Jones, and George Miranda