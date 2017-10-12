# monitoring-containerized-apps

All of these demos were done with the CoreOS Tectonic Sandbox which is a local environment for CoreOS's Enterprise Kubernetes platform. You can follow along at home with your Linux/macOS/Windows machine with this setup as well just [download Tectonic Sandbox](https://coreos.com/tectonic/sandbox).

## Setting up an app to monitor

Based on the application monitoring guide on [CoreOS Tectonic
docs](https://coreos.com/tectonic/docs/latest/tectonic-prometheus-operator/user-guides/application-monitoring.html).

```
kubectl run --image quay.io/coreos/prometheus-example-app example-app --expose --port 8080 -l app=example-app
```

## Setting up the Service Label

```
kubectl label service example-app tier=frontend
```

Visit
http://localhost:8001/api/v1/namespaces/default/services/monitor-app/proxy/

## Setting up a Service Monitor Using Labels

The ServiceMonitor targets what to monitor with label selectors.

```
apiVersion: monitoring.coreos.com/v1alpha1
kind: ServiceMonitor
metadata:
  name: frontend
  labels:
    tier: frontend
spec:
  selector:
    matchLabels:
      tier: frontend
  endpoints:
  - port: '8080'
```

The Prometheus resource creates a new Prometheus instances.

```
apiVersion: monitoring.coreos.com/v1alpha1
kind: Prometheus
metadata:
  name: frontend
  labels:
    prometheus: frontend
spec:
  version: v1.7.1
  serviceAccountName: prometheus-frontend
  serviceMonitorSelector:
    matchLabels:
      tier: frontend
  ruleSelector:
    matchLabels:
      prometheus: frontend
  resources:
    requests:
      memory: 400Mi
  alerting:
    alertmanagers:
    - namespace: tectonic-system
      name: alertmanager-main
      port: web
```

And the ServiceAccount and ClusterRoleBinding give the Prometheus instance access to the Kubernetes API.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-frontend
```

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-k8s
subjects:
- kind: ServiceAccount
  name: prometheus-frontend
  namespace: default
```

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: example-app
  labels:
    prometheus: frontend
data:
  alerting.rules: |
    ALERT HighErrorRate
      IF sum by(service, code) (http_requests_total{code="404"}) > 10
      LABELS {
        severity = "critical",
      }
      ANNOTATIONS {
        summary = "High Error Rate",
        description = "{{ $labels.service }} is experiencing high error rates.",
      }
```

Connect to the Prometheus setup on http://localhost:9090

```
kubectl get pods -l app=prometheus -o name | \
	sed 's/^.*\///' | \
	xargs -I{} kubectl port-forward {} 9090:9090
```


## Put some load on it

The boom load testing tool is fast and easy to use. Get it now:

```
go get -u github.com/rakyll/boom
```

Now put some 200 OK request load on it:

```
http://localhost:8001/api/v1/namespaces/default/services/example-app:8080/proxy/
```
