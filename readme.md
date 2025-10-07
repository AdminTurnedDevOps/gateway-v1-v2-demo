# Gloo Gateway (Gloo Edge) Demo

![](images/1.png)

The purpose of this demo is to showcase how to not only install and configure Gloo Gateway with Gloo Edge APIs, but to have comprehensive tests for:

- Resilience
- Reliability
- Failover
- Uptime

And overall traffic management needs

### CLI
```
curl -sL https://run.solo.io/gloo/install | sh
export PATH=$HOME/.gloo/bin:$PATH
```

```
glooctl install gateway
```

### Helm
```
helm repo add gloo https://storage.googleapis.com/solo-public-helm
helm repo update
```

```
helm install gloo gloo/gloo --namespace gloo-system --create-namespace
```