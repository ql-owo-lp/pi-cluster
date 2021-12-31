# Centralized Monitoring with Prometheus

Prometheus stack installation for kubernetes using Prometheus Operator can be streamlined using [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus) project maintaned by the community.

This project collects Kubernetes manifests, Grafana dashboards, and Prometheus rules combined with documentation and scripts to provide easy to operate end-to-end Kubernetes cluster monitoring with Prometheus using the Prometheus Operator.

Components included in this package:

- The Prometheus Operator
- Highly available Prometheus
- Highly available Alertmanager
- Prometheus node-exporter
- Prometheus Adapter for Kubernetes Metrics APIs
- kube-state-metrics
- Grafana

This stack is meant for cluster monitoring, so it is pre-configured to collect metrics from all Kubernetes components.

## Kube-Prometheus Stack installation

Kube-prometheus stack can be installed using helm [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) maintaind by the community

- Step 1: Add the Elastic repository:
    ```
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    ```
- Step2: Fetch the latest charts from the repository:
    ```
    helm repo update
    ```
- Step 3: Create namespace
    ```
    kubectl create namespace monitoring
    ```
- Step 3: Create values.yml for configuring VolumeClaimTemplates using longhorn and Grafana's admin password, list of plugins to be installed and disabling the monitoring of kubernetes components (Scheduler, Controller Manager and Proxy). See issue [#22](https://github.com/ricsanfre/pi-cluster/issues/22)

  ```yml
      alertmanager:
        alertmanagerSpec:
          storage:
            volumeClaimTemplate:
              spec:
                storageClassName: longhorn
                accessModes: ["ReadWriteOnce"]
                resources:
                  requests:
                    storage: 50Gi
      prometheus:
        prometheusSpec:
          storageSpec:
            volumeClaimTemplate:
              spec:
                storageClassName: longhorn
                accessModes: ["ReadWriteOnce"]
                resources:
                  requests:
                    storage: 50Gi
      grafana:
        # Admin user password
        adminPassword: "admin_password"
        # List of grafana plugins to be installed
        plugins:
          - grafana-piechart-panel
            kubeApiServer:
        enabled: true
      kubeControllerManager:
        enabled: false
      kubeScheduler:
        enabled: false
      kubeProxy:
        enabled: false
      kubeEtcd:
        enabled: false
   ```yml

- Step 3: Install kube-Prometheus-stack in the monitoring namespace with the overriden values
    ```
    helm install -f values.yml kube-prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring
    ```

## Ingress resources configuration

Enable external access to Prometheus, Grafana and AlertManager through Ingress Controller

Create a Ingress rule to make prometheus stack front-ends available through the Ingress Controller (Traefik) using a specific URLs (`prometheus.picluster.ricsanfre.com` , `grafana.picluster.ricsanfre.com` and `alertmanager.picluster.ricsanfre.com`), mapped by DNS to Traefik Load Balancer service external IP.

prometheus, Grafana and alertmanager backend are not providing secure communications (HTTP traffic) and thus Ingress resource will be configured to enable HTTPS (Traefik TLS end-point) and redirect all HTTP traffic to HTTPS.
Since prometheus frontend does not provide any authentication mechanism, Traefik HTTP basic authentication will be configured. 


- Step 1. Create a manifest file `prometheus_ingress.yml`

Two Ingress resources will be created, one for HTTP and other for HTTPS. Traefik middlewares, HTTPS redirect and basic authentication will be used. 

```yml
---
# HTTPS Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: k3s-monitoring
  annotations:
    # HTTPS as entry point
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    # Enable TLS
    traefik.ingress.kubernetes.io/router.tls: "true"
    # Use Basic Auth Midleware configured
    traefik.ingress.kubernetes.io/router.middlewares: traefik-system-basic-auth@kubernetescrd
    # Enable cert-manager to create automatically the SSL certificate and store in Secret
    cert-manager.io/cluster-issuer: self-signed-issuer
    cert-manager.io/common-name: prometheus
spec:
  tls:
  - hosts:
    - prometheus.picluster.ricsanfre.com
    secretName: prometheus-tls
  rules:
  - host: prometheus.picluster.ricsanfre.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kube-prometheus-stack-prometheus
            port:
              number: 9090

---
# http ingress for http->https redirection
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: prometheus-redirect
  namespace: k3s-monitoring
  annotations:
    # Use redirect Midleware configured
    traefik.ingress.kubernetes.io/router.middlewares: traefik-system-redirect@kubernetescrd
    # HTTP as entrypoint
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
    - host: prometheus.picluster.ricsanfre.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: kube-prometheus-stack-prometheus
              port:
                number: 9090
```
- Step 2. Create a manifest file `grafana_ingress.yml`

Two Ingress resources will be created, one for HTTP and other for HTTPS. Traefik middlewares HTTPS redirect will be used

```yml
---
# HTTPS Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: k3s-monitoring
  annotations:
    # HTTPS as entry point
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    # Enable TLS
    traefik.ingress.kubernetes.io/router.tls: "true"
    # Enable cert-manager to create automatically the SSL certificate and store in Secret
    cert-manager.io/cluster-issuer: self-signed-issuer
    cert-manager.io/common-name: grafana
spec:
  tls:
  - hosts:
    - grafana.picluster.ricsanfre.com
    secretName: grafana-tls
  rules:
  - host: grafana.picluster.ricsanfre.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kube-prometheus-stack-grafana
            port:
              number: 80

---
# http ingress for http->https redirection
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: grafana-redirect
  namespace: k3s-monitoring
  annotations:
    # Use redirect Midleware configured
    traefik.ingress.kubernetes.io/router.middlewares: traefik-system-redirect@kubernetescrd
    # HTTP as entrypoint
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
    - host: grafana.picluster.ricsanfre.com
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
- Step 3. Create a manifest file `alertmanager_ingress.yml`

Two Ingress resources will be created, one for HTTP and other for HTTPS. Traefik middlewares HTTPS redirect will be used

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: alertmanager-ingress
  namespace: k3s-monitoring
  annotations:
    # HTTPS as entry point
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    # Enable TLS
    traefik.ingress.kubernetes.io/router.tls: "true"
    # Use Basic Auth Midleware configured
    traefik.ingress.kubernetes.io/router.middlewares: traefik-system-basic-auth@kubernetescrd
    # Enable cert-manager to create automatically the SSL certificate and store in Secret
    cert-manager.io/cluster-issuer: self-signed-issuer
    cert-manager.io/common-name: alertmanager
spec:
  tls:
  - hosts:
    - alertmanager.picluster.ricsanfre.com
    secretName: prometheus-tls
  rules:
  - host: alertmanager.picluster.ricsanfre.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kube-prometheus-stack-alertmanager
            port:
              number: 9093

---
# http ingress for http->https redirection
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: alertmanager-redirect
  namespace: k3s-monitoring
  annotations:
    # Use redirect Midleware configured
    traefik.ingress.kubernetes.io/router.middlewares: traefik-system-redirect@kubernetescrd
    # HTTP as entrypoint
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
    - host: alertmanager.picluster.ricsanfre.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: kube-prometheus-stack-alertmanager
              port:
                number: 9093

``` 
- Step 4. Apply the manifest file

    kubectl apply -f prometheus_ingress.yml grafana_ingress.yml alertmanager_ingress.yml


## K3S components monitoring

In order to monitor Kubernetes components (Scheduler, Controller Manager and Proxy), default resources created by kube-prometheus-operator (headless service, service monitor and grafana dashboards) are not valid for monitoring K3S because  K3S is emitting the same metrics on the three end-points, causing prometheus to consume high memory causing worker node outage. See issue [#22](https://github.com/ricsanfre/pi-cluster/issues/22) for more details.


- Create a manifest file `k3s-metrics-service.yml` for creating the Kuberentes service used by Prometheus to scrape K3S metrics.

  This service must be a [headless service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services), for allowing Prometheus service discovery process of each of the pods behind the service. Since the metrics are exposed not by a pod but by a k3s process, the service need to be defined [`without selector`](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors) and the `endpoints` must be defined explicitely

  The service will be use the k3s-proxy endpoint (TCP port 10249) for scraping all metrics. 

  ```yml
  ---
  # Headless service for K3S metrics. No Selector
  apiVersion: v1
  kind: Service
  metadata:
    name: k3s-metrics-service
    labels:
      app: k3s-metrics
    namespace: kube-system
  spec:
    clusterIP: None
    ports:
    - name: http-metrics
      port: 10249
      protocol: TCP
      targetPort: 10249
    type: ClusterIP

  ---
  # Endpoint for the headless service without selector
  apiVersion: v1
  kind: Endpoints
  metadata:
    name: k3s-metrics-service
    namespace: kube-system
  subsets:
  - addresses:
    - ip: 10.0.0.11
    ports:
    - name: http-metrics
      port: 10249
      protocol: TCP
  ```

- Create manifest file for defining the service monitor resource for let Prometheus discover this target

  The Prometheus custom resource definition (CRD), `ServiceMonitoring` will be used to automatically discover K3S metrics endpoint as a Prometheus target.

  ```yml
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    labels:
      app: k3s
      release: kube-prometheus-stack
    name: k3s-prometheus-servicemonitor
    namespace: k3s-monitoring
  spec:
    namespaceSelector:
      matchNames:
      - kube-system
    selector:
      matchLabels:
        app: k3s-metrics
    endpoints:
      - port: http-metrics
        path: /metrics
  ```


- Apply manifest file

  kubectl apply -f k3s-metrics-service.yml k3s-servicemonitor.yml

- Check target is automatically discovered in Prometheus UI

  http://prometheus.picluster.ricsanfre/targets

### K3S Grafana dashboards

Kubernetes-controller-manager, kubernetes-proxy and kuberetes-scheduler dashboards can be donwloaded from grafana.com:

- Kube Proxy: https://grafana.com/grafana/dashboards/12129
- Kube Controller Manager: https://grafana.com/grafana/dashboards/12122
- Kube Scheduler: https://grafana.com/grafana/dashboards/12130

## Traefik Monitoring

The Prometheus custom resource definition (CRD), `ServiceMonitoring` will be used to automatically discover Traefik metrics endpoint as a Prometheus target.

- Create a manifest file `traefik-servicemonitor.yml`

```yml
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: traefik
    release: kube-prometheus-stack
  name: traefik
  namespace: k3s-monitoring
spec:
  endpoints:
    - port: traefik
      path: /metrics
  namespaceSelector:
    matchNames:
      - kube-system
  selector:
    matchLabels:
      app.kubernetes.io/instance: traefik
      app.kubernetes.io/name: traefik-dashboard

``` 
> NOTE: Important to set `label.release` to the value specified for the helm release during Prometheus operator installation (`kube-prometheus-stack`).

- Apply manifest file

  kubectl apply -f traefik-servicemonitor.yml


- Check target is automatically discovered in Prometheus UI

  http://prometheus.picluster.ricsanfre/targets

### Traefik Grafana dashboard

Traefik dashboard can be donwloaded from grafana.com: (https://grafana.com/grafana/dashboards/11462). This dashboard has as prerequisite to have installed `grafana-piechart-panel` plugin. The list of plugins to be installed can be specified during kube-prometheus-stack helm deployment as values (`grafana.plugins` variable).


## Longhorn Monitoring

As stated by official [documentation](https://longhorn.io/docs/1.2.2/monitoring/prometheus-and-grafana-setup/), Longhorn Backend service is a service pointing to the set of Longhorn manager pods. Longhorn’s metrics are exposed in Longhorn manager pods at the endpoint http://LONGHORN_MANAGER_IP:PORT/metrics.

Backend endpoint is already exposing Prometheus metrics.

The Prometheus custom resource definition (CRD), `ServiceMonitoring` will be used to automatically discover Longhorn metrics endpoint as a Prometheus target.

- Create a manifest file `longhorm-servicemonitor.yml`

```yml
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: longhorn
    release: kube-prometheus-stack
  name: longhorn-prometheus-servicemonitor
  namespace: k3s-monitoring
spec:
  selector:
    matchLabels:
      app: longhorn-manager
  namespaceSelector:
    matchNames:
    - longhorn-system
  endpoints:
  - port: manager

``` 
> NOTE: Important to set `label.release` to the value specified for the helm release during Prometheus operator installation (`kube-prometheus-stack`).

- Apply manifest file

  kubectl apply -f longhorn-servicemonitor.yml


- Check target is automatically discovered in Prometheus UI

  http://prometheus.picluster.ricsanfre/targets


```yml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: longhorn-prometheus-servicemonitor
  namespace: monitoring
  labels:
    name: longhorn-prometheus-servicemonitor
spec:
  selector:
    matchLabels:
      app: longhorn-manager
  namespaceSelector:
    matchNames:
    - longhorn-system
  endpoints:
  - port: manager

```

### Longhorn Grafana dashboard

Longhorn dashboard sample can be donwloaded from grafana.com: (https://grafana.com/grafana/dashboards/13032).


## Provisioning Dashboards automatically

Custom Grafana dashboards can be added creating CongigMap resources, containing dashboard definition in json format, because kube-prometheus-stack configure by default grafana provisioning sidecar to check for new ConfigMaps containing label `grafana_dashboard`

The default chart values are:

```yml

grafana:
  sidecar:
    dashboards:
      SCProvider: true
      annotations: {}
      defaultFolderName: null
      enabled: true
      folder: /tmp/dashboards
      folderAnnotation: null
      label: grafana_dashboard
      labelValue: null
```

Check grafana chart [documentation](https://github.com/grafana/helm-charts/tree/main/charts/grafana#sidecar-for-dashboards) explaining how to enable/use dashboard provisioning side-car.

Config Map resouce containing as data the json dashboard definition 

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sample-grafana-dashboard
  labels:
     grafana_dashboard: "1"
data:
  dashboard.json: |-
  [...]

```
Due to issue Grafana [issue](https://github.com/grafana/grafana/issues/10786), json files need to be modified before inserting them into ConfigMap yaml file, in order to detect DS_PROMETHEUS datasource. See issue [#18](https://github.com/ricsanfre/pi-cluster/issues/18)

Modify each json file adding the following code to `templating` section

```json
"templating": {
    "list": [
      {
        "hide": 0,
        "label": "datasource",
        "name": "DS_PROMETHEUS",
        "options": [],
        "query": "prometheus",
        "refresh": 1,
        "regex": "",
        "type": "datasource"
      }
```

