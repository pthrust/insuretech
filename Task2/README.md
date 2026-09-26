# Динамическое масштабирование контейнеров

## Структура Task2

```
Task2/
├── deployment.yaml              # Deployment приложения (1 реплика, 30Mi)
├── service.yaml                 # Service для доступа к приложению
├── hpa-memory.yaml              # HPA по утилизации памяти (80%, max 10)
├── hpa-rps.yaml                 # HPA по RPS через Prometheus Adapter
├── locustfile.py                # Сценарий нагрузки
├── prometheus-values.yaml       # Values для установки kube-prometheus-stack
├── prometheus-adapter-values.yaml # Values для Prometheus Adapter
├── servicemonitor.yaml          # ServiceMonitor для сбора метрик
└── README.md                    # Инструкция + скриншоты
```

# Часть 1. Динамическое масштабирование по памяти

## Запуск Minikube и metrics-server

```bash
minikube start
minikube addons enable metrics-server
minikube addons enable dashboard

kubectl apply -f deployment.yaml
kubectl get pods -l app=insuretech-app

kubectl apply -f service.yaml
kubectl port-forward svc/insuretech-app 8080:8080

kubectl apply -f hpa-memory.yaml
kubectl get hpa -w
```

## Отчет

<img src="/Task2/imgs/locust.png" alt="locust statistics" width="100%"/>

<img src="/Task2/imgs/kubectl_pods.png" alt="one pod" width="100%"/>  

<img src="/Task2/imgs/kubectl_2pods.png" alt="two pods" width="100%"/>

<img src="/Task2/imgs/kubectl_hpa.png" alt="hpa" width="100%"/>

<img src="/Task2/imgs/kubectl_replica.png" alt="replica" width="100%"/>

# Часть 2. Динамическое масштабирование по RPS через Prometheus

## Установка Prometheus Stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace -f prometheus-values.yaml

kubectl -n monitoring get pods
NAME                                                  READY   STATUS    RESTARTS   AGE
monitoring-grafana-668cb8b6b7-2qljd                   3/3     Running   0          2m56s
monitoring-kube-prometheus-operator-8796d46cf-4nbhz   1/1     Running   0          2m56s
monitoring-kube-state-metrics-6d8ffd8867-6mp2q        1/1     Running   0          2m56s
monitoring-prometheus-node-exporter-9fj7z             1/1     Running   0          2m56s
prometheus-monitoring-kube-prometheus-prometheus-0    2/2     Running   0          2m7s

kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090
```

## Установка метрик

```bash
kubectl apply -f servicemonitor.yaml
```
<img src="/Task2/imgs/prometheus.png" alt="prometheus" width="100%"/>

## Prometheus Adapter для внешних метрик

```bash
helm install prometheus-adapter prometheus-community/prometheus-adapter --namespace monitoring -f prometheus-adapter-values.yaml
```

## Настройка HPA по RPS

```
kubectl delete hpa insuretech-app-memory
kubectl apply -f hpa-rps.yaml
kubectl get hpa -w
```

###  Проверка

```bash
$ kubectl describe hpa insuretech-app-rps

Name:                                  insuretech-app-rps
Namespace:                             default
Labels:                                <none>
Annotations:                           <none>
CreationTimestamp:                     Sat, 26 Sep 2026 19:00:24 +0300
Reference:                             Deployment/insuretech-app
Metrics:                               ( current / target )
  "http_requests_per_second" on pods:  25322m / 10
Min replicas:                          1
Max replicas:                          10
Behavior:
  Scale Up:
    Stabilization Window: 0 seconds
    Select Policy: Max
    Policies:
      - Type: Percent  Value: 100  Period: 15 seconds
  Scale Down:
    Stabilization Window: 120 seconds
    Select Policy: Max
    Policies:
      - Type: Percent  Value: 25  Period: 30 seconds
Deployment pods:       10 current / 10 desired
Conditions:
  Type            Status  Reason            Message
  ----            ------  ------            -------
  AbleToScale     True    ReadyForNewScale  recommended size matches current size
  ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count from pods metric http_requests_per_second
  ScalingLimited  True    TooManyReplicas   the desired replica count is more than the maximum replica count
Events:
  Type    Reason             Age    From                       Message
  ----    ------             ----   ----                       -------
  Normal  SuccessfulRescale  5m40s  horizontal-pod-autoscaler  New size: 6; reason: pods metric http_requests_per_second above target
  Normal  SuccessfulRescale  5m25s  horizontal-pod-autoscaler  New size: 10; reason: pods metric http_requests_per_second above target
```

```bash
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_per_second" | jq .
{
  "kind": "MetricValueList",
  "apiVersion": "custom.metrics.k8s.io/v1beta1",
  "metadata": {},
  "items": [
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "insuretech-app-56f9c97d74-5l6px",
        "apiVersion": "/v1"
      },
      "metricName": "http_requests_per_second",
      "timestamp": "2026-09-26T16:11:28Z",
      "value": "288m",
      "selector": null
    },
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "insuretech-app-56f9c97d74-6m5s5",
        "apiVersion": "/v1"
      },
      "metricName": "http_requests_per_second",
      "timestamp": "2026-09-26T16:11:28Z",
      "value": "288m",
      "selector": null
    },
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "insuretech-app-56f9c97d74-bdf84",
        "apiVersion": "/v1"
      },
      "metricName": "http_requests_per_second",
      "timestamp": "2026-09-26T16:11:28Z",
      "value": "288m",
      "selector": null
    },
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "insuretech-app-56f9c97d74-bkqlp",
        "apiVersion": "/v1"
      },
      "metricName": "http_requests_per_second",
      "timestamp": "2026-09-26T16:11:28Z",
      "value": "248666m",
      "selector": null
    },
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "insuretech-app-56f9c97d74-jxml2",
        "apiVersion": "/v1"
      },
      "metricName": "http_requests_per_second",
      "timestamp": "2026-09-26T16:11:28Z",
      "value": "311m",
      "selector": null
    },
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "insuretech-app-56f9c97d74-nnj4w",
        "apiVersion": "/v1"
      },
      "metricName": "http_requests_per_second",
      "timestamp": "2026-09-26T16:11:28Z",
      "value": "311m",
      "selector": null
    },
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "insuretech-app-56f9c97d74-p68p9",
        "apiVersion": "/v1"
      },
      "metricName": "http_requests_per_second",
      "timestamp": "2026-09-26T16:11:28Z",
      "value": "288m",
      "selector": null
    },
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "insuretech-app-56f9c97d74-s52z9",
        "apiVersion": "/v1"
      },
      "metricName": "http_requests_per_second",
      "timestamp": "2026-09-26T16:11:28Z",
      "value": "311m",
      "selector": null
    },
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "insuretech-app-56f9c97d74-vmhhx",
        "apiVersion": "/v1"
      },
      "metricName": "http_requests_per_second",
      "timestamp": "2026-09-26T16:11:28Z",
      "value": "311m",
      "selector": null
    },
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "insuretech-app-56f9c97d74-z4tbb",
        "apiVersion": "/v1"
      },
      "metricName": "http_requests_per_second",
      "timestamp": "2026-09-26T16:11:28Z",
      "value": "311m",
      "selector": null
    }
  ]
}
```