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

helm install gloo glooe/gloo-ee --namespace gloo-system \
  --create-namespace \
  --set-string license_key=$EDGE_LICENSE_KEY
```

After the installation, you will see two primary services (and Pods):
1. gloo: Type ClusterIP exposing the ports 9966 (grpc-proxydebug), 9977 (grpc-xds), 9979 (wasm-cache), and 9988 (grpc-validation)
2. gateway-proxy: Type LoadBalancer exposing the ports 80, 443

```
kubectl get all -n gloo-system
NAME                                READY   STATUS    RESTARTS   AGE
pod/gateway-proxy-7886f4f95-mhk2q   1/1     Running   0          11m
pod/gloo-65c66f7dfd-rg74j           1/1     Running   0          11m

NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                                                AGE
service/gateway-proxy   LoadBalancer   34.118.239.6     34.138.86.91   80:32328/TCP,443:31458/TCP                             11m
service/gloo            ClusterIP      34.118.226.115   <none>         9977/TCP,9976/TCP,9988/TCP,9966/TCP,9979/TCP,443/TCP   11m

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gateway-proxy   1/1     1            1           11m
deployment.apps/gloo            1/1     1            1           11m
```

ArgoCD installation is available as well: https://docs.solo.io/gloo-edge/latest/installation/enterprise/#argo-cd-installation

## API/CRDs

With the Gloo Edge/Gateway is installation, you will have access to enterprise-grade APIs (Customer Resource Definitions) that you can use to extend the capabilities of Kubernetes (much like any other CRD).

![](images/2.png)