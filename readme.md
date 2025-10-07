# Gloo Gateway V1/V2 Demo

![](images/1.png)

The purpose of this demo is to showcase how to not only install and configure Gloo Gateway (v1 for portal and v2 for everything else), but to have comprehensive tests for:

- Resilience
- Reliability
- Failover
- Uptime

And overall traffic management needs

## Installation Gloo Gateway v2

### Helm
```
export GLOO_GATEWAY_LICENSE_KEY=<gloo-gateway-license-key>
export AGENTGATEWAY_LICENSE_KEY=<agentgateway-license-key>
```

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml --context=$CLUSTER1
```

```
helm upgrade -i gloo-gateway-crds oci://us-docker.pkg.dev/solo-public/gloo-gateway/charts/gloo-gateway-crds \
--create-namespace \
--namespace gloo-system \
--version 2.0.0-rc.2
```

```
helm upgrade -i gloo-gateway oci://us-docker.pkg.dev/solo-public/gloo-gateway/charts/gloo-gateway \
-n gloo-system \
--version 2.0.0-rc.2 \
--set agentgateway.enabled=true \
--set licensing.glooGatewayLicenseKey=$GLOO_GATEWAY_LICENSE_KEY \
--set licensing.agentgatewayLicenseKey=$AGENTGATEWAY_LICENSE_KEY
```

With the free installation, you will see the following:

FILL IN!!!!


#### Argo/GitOps

ArgoCD installation is available as well: https://docs.solo.io/gateway/2.0.x/install/argocd/

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