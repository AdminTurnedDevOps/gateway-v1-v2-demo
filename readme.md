# Gloo Gateway V1/V2 Demo
<p align="center">
 <img src="images/1.png?raw=true" alt="Logo" width="50%" height="50%" />
</p>


The purpose of this demo is to showcase how to not only install and configure Gloo Gateway (v1 for portal and v2 for everything else), but to have comprehensive tests for:

- Resilience
- Reliability
- Failover
- Uptime

And overall traffic management needs

# Gloo Gateway V2 Configs

The following sections will showcase everything from a Gloo Gateway V2 perspective including installation, configuration, and resilience configs.

## Installation Gloo Gateway v2

### Cluster

For the purposes of this demo, you can use any managed Kubernetes cluster you'd like, but if you need a ready-to-go config, you can deploy GKE via the Terraform configs in this repo.

If you use the Terraform configs, remember to be authenticated on your local terminal to GCP.


1. `cd` into the **ggv2/gke1** directory.

2. Update the variables in `variables.tf` to reflect your variables (or create a `terraform.tfvars` file)

3. Initialize the TF config
```
terraform init
```

4. Run the plan
```
terraform plan
```

5. Apply the TF configs to create the GKE cluster with auto-approve
```
terraform apply --auto-approve
```

<p align="center">
 <img src="images/2.png?raw=true" alt="Logo" width="50%" height="50%" />
</p>


### Helm
1. Configure product key env variables

```
export GLOO_GATEWAY_LICENSE_KEY=
export AGENTGATEWAY_LICENSE_KEY=
```

2. Install Kubernetes Gateway API
You need the experimental version as Gloo Gateway v2 has a requirement of the `BackendConfigPolicy` object, which is an experimental feature in Kubernetes Gateway API.

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/experimental-install.yaml
```

3. Install Gloo Gateway v2 CRDs
```
helm upgrade -i gloo-gateway-crds oci://us-docker.pkg.dev/solo-public/gloo-gateway/charts/gloo-gateway-crds \
--namespace gloo-system \
--version 2.0.0-rc.2 \
--create-namespace
```

4. Install Gloo Gateway v2
```
helm upgrade -i gloo-gateway oci://us-docker.pkg.dev/solo-public/gloo-gateway/charts/gloo-gateway \
-n gloo-system \
--version 2.0.0-rc.2 \
--set licensing.glooGatewayLicenseKey=$GLOO_GATEWAY_LICENSE_KEY \
```


#### Argo/GitOps

ArgoCD installation is available as well: https://docs.solo.io/gateway/2.0.x/install/argocd/

## Sample App Deployment

1. Create the Namespace for the microapp (extensive decoupled app)
```
kubectl create ns microapp
```

2. Deploy the sample decoupled application stack
```
kubectl apply -f ggv2/sampleapp-microdemo/microservices-demo/release/kubernetes-manifests.yaml -n microapp
```

3. Confirm that the app stack is running
```
kubectl get pods -n microapp
```

You can also see the Services that are deployed, which is what you'll use to create the backend routes in the next step.

```
kubectl get svc -n microapp
```

4. Create a Gateway for the application

```
kubectl apply --context=$CLUSTER1 -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: frontend-gateway
  namespace: microapp
spec:
  gatewayClassName: gloo-gateway-v2
  listeners:
  - name: frontend
    port: 80
    protocol: HTTP
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend
  namespace: microapp
spec:
  parentRefs:
  - name: frontend-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
      - name: frontend
        port: 80
EOF
```

5. Check to see the gateway IP address.

```
kubectl get gateway -n microapp
```

Example output below:
```
NAME               CLASS             ADDRESS         PROGRAMMED   AGE
frontend-gateway   gloo-gateway-v2   x.x.x.x   True         36m
```

![](images/3.png)

## Gateway UI
1. Capture your cluster name as an environment variable for the UI installation in the coming steps.
```
export CLUSTER_NAME=fegatewayv1

echo $CLUSTER_NAME
```

2. Add the Gloo Platform Chart (for the UI)
```
helm repo add gloo-platform https://storage.googleapis.com/gloo-platform/helm-charts
helm repo update
```

3. Install the Gloo Platform CRDs
```
helm upgrade -i gloo-platform-crds gloo-platform/gloo-platform-crds \
--namespace=gloo-system \
--version=2.10.1 \
--set installEnterpriseCrds=false
```

4. Deploy the UI Helm Chart

You'll see that the enterprise version of the chart gives you:
- The UI
- Gloo Insights that you can see from the portal
- Prometheus enabled for metrics collection
- A Telemetry collector

```
helm upgrade -i gloo-platform gloo-platform/gloo-platform \
--namespace gloo-system \
--version=2.10.1 \
-f - <<EOF
common:
  adminNamespace: "gloo-system"
  cluster: $CLUSTER_NAME
glooInsightsEngine:
  enabled: true
glooAnalyzer:
  enabled: true
glooUi:
  enabled: true
licensing:
  glooGatewayLicenseKey: $GLOO_GATEWAY_LICENSE_KEY
prometheus:
  enabled: true
telemetryCollector:
  enabled: true
  mode: deployment
  replicaCount: 1
EOF
```

5. Ensure that the UI it's running as expected (you should see 3 containers in the Pod)
```
kubectl get pods -n gloo-system
```

6. Access the UI
```
kubectl port-forward deployment/gloo-mesh-ui -n gloo-system 8090
```

![](images/4.png)

## Monitoring, Observability, & Telemetry

1. Add the Grafana Helm Chart
```
helm repo add grafana https://grafana.github.io/helm-charts
```

2. Install Grafana in the `monitoring` Namespace
```
helm install grafana grafana/grafana --namespace monitoring --create-namespace
```

3. Retrieve the default admin password from the Kubernetes Secret.

The default username is: `admin`

```
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

4. Access the Grafana UI
```
kubectl port-forward svc/grafana 3000:80 --namespace monitoring
```

5. Add a new metrics endpoint by going to: **Connections > Data Sources > Choose Prometheus

Under the Connection URL, add the following:
```
http://prometheus-server.gloo-system.svc.cluster.local:80
```

You'll now be able to see Metrics and create Dashboard in Grafana

![](images/6.png)

**ADD IN LIVE TRAFFIC DEBUGGING**
**ADD IN more observability pieces**

https://docs.solo.io/gateway/2.0.x/observability/

## Rate Limiting

WIP

## Advanced Routing

WIP

- Load balanced routing
- Canary
- Authenication
- Redirection

https://docs.solo.io/gateway/2.0.x/traffic-management/

## Resiliency

WIP

https://docs.solo.io/gateway/2.0.x/resiliency/

## Cleanup
To prepare your environment for the next part of the demo, which will be on Gloo Gateway v1 with Portal, destroy your cluster.

If you use the GKE config within this repo:

1. `cd` into the **ggv2/gke1** directory.

2. Destroy the cluster
```
terraform destroy --auto-approve
```

# Gloo Gateway V1 (Portal)
In this section, you will find the full configuration for setting up Gloo Gateway v1. The reason why v1 will be used is because Portal will not be GA until Gloo Gateway v2.2.

## Installation Gateway v1

### Cluster

For the purposes of this demo, you can use any managed Kubernetes cluster you'd like, but if you need a ready-to-go config, you can deploy GKE via the Terraform configs in this repo.

If you use the Terraform configs, remember to be authenticated on your local terminal to GCP.


1. `cd` into the **ggv2/gke1** directory.

2. Update the variables in `variables.tf` to reflect your variables (or create a `terraform.tfvars` file)

3. Initialize the TF config
```
terraform init
```

4. Run the plan
```
terraform plan
```

5. Apply the TF configs to create the GKE cluster with auto-approve
```
terraform apply --auto-approve
```

<p align="center">
 <img src="images/2.png?raw=true" alt="Logo" width="50%" height="50%" />
</p>

### Helm

1. Set your Gloo License Key variable
```
export GLOO_GATEWAY_LICENSE_KEY=
```

2. Install Kubernetes Gateway API
```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
```

3. Add the Gloo Gateway v1 Enterprise Helm repo
```
helm repo add glooe https://storage.googleapis.com/gloo-ee-helm
helm repo update
```

4. Install GGv1 Enterprise

The installation also includes:
- Grafana
- Prometheus

```
helm upgrade --install -n gloo-system gloo glooe/gloo-ee \
--create-namespace \
--version 1.20.1 \
--set-string license_key=$GLOO_GATEWAY_LICENSE_KEY \
-f - <<EOF
gloo:
  gatewayProxies:
    gatewayProxy:
      disabled: true
  kubeGateway:
    enabled: true
    portal:
      enabled: true
  gloo:
    disableLeaderElection: true
gloo-fed:
  enabled: false
  glooFedApiserver:
    enable: false
grafana:
  defaultInstallationEnabled: false
observability:
  enabled: false
prometheus:
  enabled: false
EOF
```

5. Ensure that Gloo Gateway v1 is running as expected
```
kubectl get pods -n gloo-system
```

6. Retrieve the Gateway Class 
```
kubectl get gatewayclass gloo-gateway
```

## Portal

In this previous section, you installed Gloo Gateway v1. If you take a look at the Helm config, the `portal: true` was already added in, so you won't need to do any extra configuration for the installation of Portal.

### Installation Of Portal

1. Ensure Portal is up and operational
```
kubectl get pods -n gloo-system -l app=gateway-portal-web-server
```

You should see an output similar to the below:
```
NAME                                        READY   STATUS    RESTARTS   AGE
gateway-portal-web-server-c9c78db5b-dfpfm   1/1     Running   0          2m1s
```

If you have `glooctl` installed, you can also check to confirm that all implementations (xDS, Gateways, Proxies, Rate Limiting Server, etc.) is ready to go.

```
glooctl check

glooctl binary version (1.19.6) differs from server components (v1.20.1) by at least a minor version.
Consider running:
glooctl upgrade --release=v1.20.1
----------

Checking Deployments... OK
Checking Pods... OK
Checking Upstreams... OK
Checking UpstreamGroups... OK
Checking AuthConfigs... OK
Checking RateLimitConfigs... OK
Checking VirtualHostOptions... OK
Checking RouteOptions... OK
Checking Secrets... OK
Checking VirtualServices... OK
Checking Gateways... OK
Checking Proxies... OK
No active gateway-proxy pods exist in cluster
Checking xds metrics... OK
Checking rate limit server... OK

Detected Kubernetes Gateway integration!
Checking Kubernetes GatewayClasses... OK
Checking Kubernetes Gateways... OK
Checking Kubernetes HTTPRoutes... OK

Skipping Gloo Instance check -- Gloo Federation not detected.
No problems detected.
```

### Deploy Sample Apps

For this section, please use the docs below and deploy the **Tracks** sample app

https://docs.solo.io/gateway/latest/portal/tutorials/setup/#apps

### Create API Products

For this section, please use the **Tracks** section as that's the demo app you deployed in the previous section

https://docs.solo.io/gateway/latest/portal/tutorials/portal/

### Create A Portal

https://docs.solo.io/gateway/latest/portal/tutorials/apis/

## Agentgateway (GGv2)

agentgateway is a way to secure, observe, and control the traffic to and from LLMs and MCP servers.

This wouldn't be part of a standard Gloo Gateway POC, but if interesting in seeing it, it's a "nice to see".

### Helm
1. Configure product key env variables

```
export GLOO_GATEWAY_LICENSE_KEY=
export AGENTGATEWAY_LICENSE_KEY=
```

2. Install Kubernetes Gateway API
You need the experimental version as Gloo Gateway v2 has a requirement of the `BackendConfigPolicy` object, which is an experimental feature in Kubernetes Gateway API.

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/experimental-install.yaml
```

3. Install Gloo Gateway v2 CRDs
```
helm upgrade -i gloo-gateway-crds oci://us-docker.pkg.dev/solo-public/gloo-gateway/charts/gloo-gateway-crds \
--namespace gloo-system \
--version 2.0.0-rc.2 \
--create-namespace
```

4. Install Gloo Gateway v2
```
helm upgrade -i gloo-gateway oci://us-docker.pkg.dev/solo-public/gloo-gateway/charts/gloo-gateway \
-n gloo-system \
--version 2.0.0-rc.2 \
--set gateway.aiExtension.enabled=true \
--set agentgateway.enabled=true \
--set licensing.glooGatewayLicenseKey=$GLOO_GATEWAY_LICENSE_KEY \
--set licensing.agentgatewayLicenseKey=$AGENTGATEWAY_LICENSE_KEY
```


## Helpful Docs

Based on the Evaluation Criteria doc, the following links will be of help for knowing that Gloo Gateway supports what you need.

1. JWT/Payload support: https://docs.solo.io/gateway/2.0.x/security/jwt/overview/
2. Load balancing support: https://docs.solo.io/gateway/2.0.x/traffic-management/session-affinity/loadbalancing/
3. Kubernetes CRD (API/object/kind) support: https://docs.solo.io/gateway/2.0.x/reference/api/gloo-gateway/
4. Gloo Mesh/Gloo Gateway working together in multi-cluster support: https://github.com/AdminTurnedDevOps/ambient-mesh-lite-demo/blob/main/multi-cluster/sampleapp-microdemo/setup.md
5. Canary deployments with Mirroring: https://docs.solo.io/gateway/2.0.x/resiliency/mirroring/