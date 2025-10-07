# Gloo Gateway (Gloo Edge) Demo

![](images/1.png)

The purpose of this demo is to showcase how to not only install and configure Gloo Gateway with Gloo Edge APIs, but to have comprehensive tests for:

- Resilience
- Reliability
- Failover
- Uptime

And overall traffic management needs

## Installation

### CLI
```
curl -sL https://run.solo.io/gloo/install | sh
export PATH=$HOME/.gloo/bin:$PATH
```

```
export EDGE_LICENSE_KEY=
glooctl install gateway enterprise --license-key $EDGE_LICENSE_KEY
```

### Helm
```
helm repo add glooe https://storage.googleapis.com/gloo-ee-helm

helm repo update
```

```
export EDGE_LICENSE_KEY=

helm upgrade --install gloo glooe/gloo-ee --namespace gloo-system \
  --create-namespace \
  --set-string license_key=$EDGE_LICENSE_KEY
```

With the free installation, you will see two primary services (and Pods):

1. gloo: Type ClusterIP exposing the ports 9966 (grpc-proxydebug), 9977 (grpc-xds), 9979 (wasm-cache), and 9988 (grpc-validation)
2. gateway-proxy: Type LoadBalancer exposing the ports 80, 443

```
kubectl get all -n gloo-system
NAME                                READY   STATUS    RESTARTS   AGE
pod/gateway-proxy-7886f4f95-mhk2q   1/1     Running   0          11m
pod/gloo-65c66f7dfd-rg74j           1/1     Running   0          11m
```

However, with Enterprise you will get the following out of the box:
- External Auth
- Prometheus
- Grafana
- Envoy-integrated rate limiting
- Caching

### UI

You can access the Gloo Edge/Gateway UI by doing the following:
```
kubectl port-forward svc/gloo-fed-console -n gloo-system 8090:809
```

#### Argo/GitOps

ArgoCD installation is available as well: https://docs.solo.io/gloo-edge/latest/installation/enterprise/#argo-cd-installation

## API/CRDs

With the Gloo Edge/Gateway is installation, you will have access to enterprise-grade APIs (Customer Resource Definitions) that you can use to extend the capabilities of Kubernetes (much like any other CRD).

These extensive and mature APIs will give you the ability to manage/route traffic, set policies, and configure authentication.

![](images/2.png)

## Sample App Deployment

1. Create the Namespace for the microapp (extensive decoupled app)
```
kubectl create ns microapp
```

2. Deploy the sample decoupled application stack
```
kubectl apply -f sampleapp-microdemo/microservices-demo/release/kubernetes-manifests.yaml -n microapp
```

3. Confirm that the app stack is running
```
kubectl get pods -n microapp
```

4. Create a Gateway for the application

```
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: frontend-gateway
  namespace: microapp
spec:
  gatewayClassName: istio
  listeners:
  - name: frontend
    port: 80
    protocol: HTTP
EOF
```

## Gloo Gateway/Edge Portal

1. Add the Gloo Portal Helm Chart
```
helm repo add gloo-portal https://storage.googleapis.com/dev-portal-helm

helm repo update
```

2. Install the portal (consuming the current Gloo Edge/Gateway license key)
```
helm upgrade --install gloo-portal gloo-portal/gloo-portal \
-n gloo-portal \
--create-namespace \
-f - <<EOF
glooEdge:
  enabled: true
licenseKey:
  secretRef:
    name: license
    namespace: gloo-system
    key: license-key
EOF
```

3. Ensure that all resources are up and running as expected.
```
kubectl get all -n gloo-portal
```

4. To reach the portal (pending Portal configurations)
```
kubectl port-forward -n gloo-portal svc/gloo-portal-admin-server 8080:8080
```

## Monitoring, Observability, & Telemetry