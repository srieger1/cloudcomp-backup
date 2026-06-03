# Introduction to monitoring using kube-prometheus-stack

> [!CAUTION]
> When deploying kube-prometheus-stack along with taskflow and ArgoCD, most likely your memory and CPU resources of the RKE2 deployment with a single worker in OpenStack will be exhausted. You can simply change the number of workers from 1 to 2 or even 3. If you do not have other things running in OpenStack, this will still fit in your quota. If you need more resources, you can contact the tutors or the professor to get slightly more. After increasing the amount of workers as in `main.tf` in `examples/hs-fulda` directory, you need to run `terraform apply` and wait for the additional worker to get installed. It will be reported in `kubectl get nodes` afterwards. The `terraform-openstack-rke2` repo we use, will also make sure, that workers are automatically placed on different hardware nodes using a server group in OpenStack that has Anti-Affinity (instances should be separated on different nodes, Affinity would be the opposite if you want to make sure that instances are running on the same hardware).

A popular artifact to deploy monitoring on k8s is the [kube-prometheus-stack](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack).

You can install it by using the `kube-prometheus-stack-values.yaml` provided in this repo, that exposes grafana to use an exernal IP (serviceType: LoadBalancer again) and also sets up scraping for RKE2 which is by default only accessing secure connections to scrape metrics. If you do not use RKE2, some students use k8s in Docker Desktop, k3d, k3s etc., then remove the custom values.

```bash
helm upgrade --install kube-prometheus-stack oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack \
  -n monitoring --create-namespace \
  -f kube-prometheus-stack-values.yaml
```

you can also use our local harbor registry

```bash
helm upgrade --install kube-prometheus-stack oci://harbor.cs.hs-fulda.de/ghcr.io/prometheus-community/charts/kube-prometheus-stack \
  -n monitoring --create-namespace \
  -f kube-prometheus-stack-values.yaml
```

Natively, Rancher and RKE2 come with their own [Monitoring and Alerting](https://ranchermanager.docs.rancher.com/integrations-in-rancher/monitoring-and-alerting), which is in production the better option, but also limited to Rancher. The solution above is supported by all major k8s distributions.

## Add Loki and Alloy for logs

```bash
helm repo add common-charts https://rishang.github.io/helm-chart/charts
helm repo update
helm install -f loki-stack-values.yaml loki-stack common-charts/loki-stack --namespace monitoring --create-namespace
```
