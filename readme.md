# Gloo Gateway (Gloo Edge) Demo

![](images/1.png)

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