
```
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" INSTALL_K3S_EXEC="server --disable=traefik" sh -

 export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

### Deploy demo application which deploys 4 pods
```kubectl apply -f demo-app.yaml```

```kubectl port-forward service/frontend -n kuma-demo 8080```

> This marketplace application is currently running WITHOUT Kuma. So all traffic is going directly between the services, and not routed through any data plane proxies.

### Install kumactl and Kuma Control Plane
```
curl -L https://kuma.io/installer.sh | VERSION=2.5.0 sh -
cd kuma-2.5.0/bin
PATH=$(pwd):$PATH
cd ../..

kumactl install control-plane \
  --set "controlPlane.mode=standalone" \
  | kubectl apply -f -
```
###### Access the front-end UI
Port-forward the kuma-control-plane service in the kuma-system namespace and access the UI here: http://localhost:5681/gui/
> Note that the above app yaml has the sidecar injection enabled on the namespace kuma-demo 

```kubectl port-forward service/kuma-control-plane -n kuma-system 5681```

###### Check the pods are up and running by getting all pods in the kuma-system namespace
 ``` kubectl get pods -n kuma-system ```

Check that mtls is off, on GUI mesh object or below command
``` kubectl get meshes ```

Delete pods to inject sidecar
```kubectl delete pods --all -n kuma-demo```
### Prometheus and Grafana

```kumactl install observability | kubectl apply -f -```
```kubectl get pods -n mesh-observability```
```kubectl -n mesh-observability port-forward po/grafana-78d497dcf9-9dnz7 3000```
#### Configure mesh object to send metrics
Kuma facilitates consistent traffic metrics across all data plane proxies in your mesh
```
cat <<EOF | kubectl apply -f - 
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  metrics:
    enabledBackend: prometheus-1
    backends:
    - name: prometheus-1
      type: prometheus
EOF
```

###### enable mTLS using below command:
```
cat <<EOF | kubectl apply -f - 
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  mtls:
    enabledBackend: ca-1
    backends:
    - name: ca-1
      type: builtin
  metrics:
    enabledBackend: prometheus-1
    backends:
    - name: prometheus-1
      type: prometheus
EOF
```

Will see no diff as there is a default-allow all traffic permission. 

```kubectl delete trafficpermission -n kuma-demo --all ```
#### Adding granular permissions
```
cat <<EOF | kubectl apply -f -
apiVersion: kuma.io/v1alpha1
kind: TrafficPermission
mesh: default
metadata:
  namespace: kuma-demo
  name: kong-to-frontend
spec:
  sources:
  - match:
      kuma.io/service: kong-validation-webhook_kuma-demo_svc_443
  destinations:
  - match:
      kuma.io/service: frontend_kuma-demo_svc_8080
---
apiVersion: kuma.io/v1alpha1
kind: TrafficPermission
mesh: default
metadata:
  namespace: kuma-demo
  name: frontend-to-backend
spec:
  sources:
  - match:
      kuma.io/service: frontend_kuma-demo_svc_8080
  destinations:
  - match:
      kuma.io/service: backend_kuma-demo_svc_3001
---
apiVersion: kuma.io/v1alpha1
kind: TrafficPermission
mesh: default
metadata:
  namespace: kuma-demo
  name: backend-to-postgres
spec:
  sources:
  - match:
      kuma.io/service: backend_kuma-demo_svc_3001
  destinations:
  - match:
      kuma.io/service: postgres_kuma-demo_svc_5432
EOF
 ```
In the first one, we allow the Kong service to communicate to the frontend. In the second one, we allow the frontend to communicate with the backend. And in the last one, we allow the backend to communicate with PostgreSQL. By not providing any permissions to Redis, traffic won't be allowed to that service
```kubectl get trafficpermissions```
Reviews won't work now since no traffic allowed to redis which can be enabled as below:
```cat <<EOF | kubectl apply -f - 
apiVersion: kuma.io/v1alpha1
kind: TrafficPermission
mesh: default
metadata:
  namespace: kuma-demo
  name: backend-to-redis
spec:
  sources:
  - match:
      kuma.io/service: backend_kuma-demo_svc_3001
  destinations:
  - match:
      kuma.io/service: redis_kuma-demo_svc_6379
EOF
```

##### Adding a traffic metric policy
Allow the traffic from Grafana to Prometheus Server and from Prometheus Server to Dataplane metrics and for other Prometheus components
```
cat <<EOF | kubectl apply -f - 
apiVersion: kuma.io/v1alpha1
kind: TrafficPermission
mesh: default
metadata:
  name: metrics-permissions
spec:
  sources:
    - match:
       kuma.io/service: prometheus-server_mesh-observability_svc_80
  destinations:
    - match:
       kuma.io/service: dataplane-metrics
    - match:
       kuma.io/service: "prometheus-alertmanager_mesh-observability_svc_80"
    - match:
       kuma.io/service: "prometheus-kube-state-metrics_mesh-observability_svc_80"
    - match:
       kuma.io/service: "prometheus-kube-state-metrics_mesh-observability_svc_81"
    - match:
       kuma.io/service: "prometheus-pushgateway_mesh-observability_svc_9091"
---
apiVersion: kuma.io/v1alpha1
kind: TrafficPermission
mesh: default
metadata:
  name: grafana-to-prometheus
spec:
   sources:
   - match:
      kuma.io/service: "grafana_mesh-observability_svc_80"
   destinations:
   - match:
      kuma.io/service: "prometheus-server_mesh-observability_svc_80"
EOF
```
### Traffic Routing
```
                        ----> backend-v0  :  service=backend, version=v0, env=prod
                      /
(browser) -> frontend   ----> backend-v1  :  service=backend, version=v1, env=intg
                      \
                        ----> backend-v2  :  service=backend, version=v2, env=dev
```
##### Scale Replicas
Backend-v1 and backend-v2 were deployed with 0 replicas so let's scale them up to one replica to see how traffic routing works:
```kubectl scale deployment kuma-demo-backend-v1 -n kuma-demo --replicas=1```

```kubectl scale deployment kuma-demo-backend-v2 -n kuma-demo --replicas=1```
Check all the pods are running like this:
``` kubectl get pods -n kuma-demo```

##### Limit traffic to v2. 80% to v0, 20% to v1
```
cat <<EOF | kubectl apply -f -
apiVersion: kuma.io/v1alpha1
kind: TrafficRoute
metadata:
  name: frontend-to-backend
  namespace: kuma-demo
mesh: default
spec:
  sources:
  - match:
      kuma.io/service: frontend_kuma-demo_svc_8080
  destinations:
  - match:
      kuma.io/service: backend_kuma-demo_svc_3001
  conf:
    split:
    - weight: 80
      destination:
        kuma.io/service: backend_kuma-demo_svc_3001
        version: v0
    - weight: 20
      destination:
        kuma.io/service: backend_kuma-demo_svc_3001
        version: v1
    - weight: 0
      destination:
        kuma.io/service: backend_kuma-demo_svc_3001
        version: v2
EOF
```
Above can be verified by refresing front-end page , the two sale (v2) will never come up

### Health Check
The goal of Health Checks is to minimize the number of failed requests due to temporary unavailability of a target endpoint. By applying a Health Check policy you effectively instruct a data plane proxy to keep track of health statuses for target endpoints. Dataplane will never send a request to an endpoint that is considered "unhealthy".
```
cat <<EOF | kubectl apply -f -
apiVersion: kuma.io/v1alpha1
kind: HealthCheck
metadata:
  name: frontend-to-backend
  namespace: kuma-demo
mesh: default
spec:
  sources:
  - match:
      kuma.io/service: frontend_kuma-demo_svc_8080
  destinations:
  - match:
      kuma.io/service: backend_kuma-demo_svc_3001
  conf:
    interval: 10s
    timeout: 2s
    unhealthyThreshold: 3
    healthyThreshold: 1
EOF
```

>Visualize on Grafana


