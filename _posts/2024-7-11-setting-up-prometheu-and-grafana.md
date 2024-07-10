---
layout: post
title: "Setting up Prometheus and Grafana on Kubernetes: A Beginner’s Guide"
---

In this blog post, I'll share my experience on setting up a monitoring stack with Prometheus and Grafana on Kubernetes. This setup assumes you have:

- Basic knowledge of Kubernetes and its components. If you're new to Kubernetes, I recommend starting with the [official Kubernetes basics tutorial](https://kubernetes.io/docs/tutorials/kubernetes-basics/).
- A Kubernetes cluster with ingress set up, either on the cloud (e.g., AWS EKS) or locally using Minikube. For AWS EKS setup, refer to the [Amazon EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html), and for Minikube, check the [Minikube Getting Started Guide](https://minikube.sigs.k8s.io/docs/start/).
- Helm (version 3.14 or higher) installed. See the [Helm installation guide](https://helm.sh/docs/intro/install/).
- kubectl (version 1.25 or higher) installed. Installation instructions can be found on the [official kubectl installation page](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

### Deploying the Monitoring Stack

We'll be using the kube-prometheus-stack, a collection of Kubernetes resources essential for setting up monitoring. The components include:

- **Prometheus:** For collecting and storing metrics as per PromQL.
- **Grafana:** For visualizing Prometheus metrics.
- **kube-state-metrics:** Exports metrics from the Kubernetes API.
- **Prometheus Node Exporter:** Exports hardware and OS metrics exposed by *NIX kernels.

These components are all packaged together in the kube-prometheus-stack helm chart, which can be found in its [GitHub repository](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) and is part of the larger [kube-prometheus project](https://github.com/prometheus-operator/kube-prometheus).

### Installation Instructions

1. **Add the Helm repository:**
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   ```

2. **Install the stack:**
   It's recommended to use a separate namespace for monitoring tools:
   ```bash
   helm install [RELEASE_NAME] prometheus-community/kube-prometheus-stack --namespace monitoring
   ```

3. **Configure custom values:**
   Customize the installation by modifying the `values.yaml` file. Here are some suggested values:
   ```yaml
   grafana:
    defaultDashboardsEditable: false
    initChownData:
      enabled: false
    serviceMonitor:
      labels:
        release: kube-prometheus-stack
    resources:
      limits:
        cpu: 300m
        memory: 400Mi
      requests:
        cpu: 200m
        memory: 300Mi
    persistence:
      enabled: true
      type: sts
      storageClassName: gp2
      accessModes:
        - ReadWriteOnce
      size: 128Gi
      finalizers:
        - kubernetes.io/pvc-protection
    adminPassword: "some-secret-password"

   kubeControllerManager:
    enabled: false

   kubeEtcd:
     enabled: false

   kubeScheduler:
     enabled: false

   kube-state-metrics:
     resources:
       limits:
         cpu: 50m
         memory: 120Mi
       requests:
         cpu: 20m
         memory: 70Mi
     prometheus:
       monitor:
         relabelings:
           - sourceLabels: [__meta_kubernetes_pod_container_name]
             targetLabel: container
             replacement: $1
             action: replace
           - sourceLabels: [__meta_kubernetes_pod_container_image]
             targetLabel: image
             replacement: $1
             action: replace

   prometheusOperator:
     resources:
       limits:
         cpu: 300m
         memory: 500Mi
       requests:
         cpu: 200m
         memory: 400Mi

   prometheus:
     serviceMonitor:
       relabelings:
         - targetLabel: cluster
           replacement: "ergeon-eks-staging"
           action: replace
     prometheusSpec:
       retention: 30d
       resources:
         limits:
           cpu: 500m
           memory: "1700Mi"
         requests:
           cpu: 200m
           memory: "1500Mi"
       storageSpec:
         volumeClaimTemplate:
           spec:
             storageClassName: gp2
             accessModes: ["ReadWriteOnce"]
             resources:
               requests:
                 storage: 256Gi

   prometheus-node-exporter:
     resources:
       limits:
         cpu: 100m
         memory: 100Mi
       requests:
         cpu: 50m
         memory: 50Mi
   ```

   Save these configurations in a file named `custom_values.yaml` and apply them during installation:
   ```bash
   helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring -f custom_values.yaml
   ```

4. **Verify the installation:**
   Check the status of the deployed pods:
   ```bash
   kubectl get pods -n monitoring
   ```

### Setting Up Ingress

To access Grafana through a web browser, set up an ingress resource. Below is an example of an ingress configuration using the NGINX ingress controller:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-monitoring
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts:
      - "monitoring.your-domain.com"
      secretName: monitoring-secret
  rules:
    - host: "monitoring.your-domain.com"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kube-prometheus-stack-grafana
                port:
                  number: 80
```
Apply this configuration using:
```bash
kubectl apply -f ingress.yaml
```

### Accessing Grafana

Once the ingress is successfully deployed, access your Grafana dashboard by navigating to `https://monitoring.your-domain.com`. Log in using the admin credentials set in the `values.yaml` file. The default username is `admin`, and the password will be the one you set as `some-secret-password`.

### Visualizing Metrics

After logging in, you will be redirected to the Grafana dashboard, where you can:

- **Explore Metrics:** Use the Explore tab to query the data stored by Prometheus and visualize metrics.
- **Manage Dashboards:** You can create new dashboards or import existing ones to better visualize various metrics.
- **Configure Data Sources:** Ensure that Prometheus is set as the default data source so that Grafana can fetch and display metrics seamlessly.
- **User Management:** Update permissions, add or remove users, and customize their roles within Grafana.

### Next Steps

To further enhance your monitoring capabilities, consider implementing the following:

- **Import Custom Dashboards:** Grafana allows you to import dashboards which can be found on [Grafana Labs' official site](https://grafana.com/grafana/dashboards). These dashboards are tailored for various use cases and data sources.
- **Third-Party Metrics Exporters:** Integrate exporters for application-level metrics, such as those for databases, hardware, or custom applications. This enriches the insights you can gain from your cluster.
- **Alerting:** Set up alerts in Grafana or Prometheus to be notified about critical conditions. Alertmanager can handle alerts sent by client applications such as the Prometheus server.
- **Horizontal Pod Autoscaling (HPA):** Utilize Prometheus metrics to automatically scale your applications based on observed metrics like CPU usage or custom metrics.

### Conclusion

Setting up Prometheus and Grafana on Kubernetes using Helm charts is a straightforward way to get a robust monitoring solution up and running. This setup not only helps in observing the state of your clusters but also aids in proactive management through metrics analysis.
