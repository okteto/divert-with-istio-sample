# Sample for Divert with Istio
This repo is a sample about using divert with istio.
The following diagram explains the architectture of the application we will be using in this sample:

![sample-app](images/sample-app.png)

### Cluster Admin tasks

The admin of the cluster must run the following commands to install Istio in the Okteto cluster:

```
helm install --namespace istio-system --create-namespace istio-base istio/base --version 1.13.8 --wait
helm install --namespace istio-system istiod istio/istiod --values istio/istiod-helm-values.yaml --version 1.13.8 --wait
kubectl create namespace istio-ingress
kubectl label namespace istio-ingress istio-injection=enabled
helm install --namespace istio-ingress istio-ingress istio/gateway --values istio/istio-ingress-helm-values.yaml --version 1.13.8 --wait
```

Replace `<<okteto-subdomain>>` in `istio-ingress-config.yaml` by the `subdomain` field in your okteto Helm values file:

```
kubectl apply -n istio-ingress -f istio/istio-ingress-config.yaml
```

### Develop tasks

Each developer must run the following commands to deploy their dev environment

### Configure Okteto CLI and kubectl credentials

```
okteto ctx use https://okteto.<<okteto-subdomain>>
okteto kubeconfig
```

### Deploy services

Then, run `okteto deploy -f okteto-staging.yml --var OKTETOSUBDOMAIN=<<your-okteto-subdomain>>` to deploy the application in the `staging` namespace.
### Test services

```
# Ingress endpoints
curl -k https://service-a-<<okteto-namespace>>.<<okteto-subdomain>>/
curl -k https://service-b-<<okteto-namespace>>.<<okteto-subdomain>>/
curl -k https://service-echo-<<okteto-namespace>>.<<okteto-subdomain>>/
curl -k https://service-a-<<okteto-namespace>>.<<okteto-subdomain>>/call-b
curl -k https://service-a-<<okteto-namespace>>.<<okteto-subdomain>>/call-echo

# curl service-b from inside service-a via the Istio sidecar (service-b.staging service entry)
kubectl -n staging exec svc/service-a -- curl -s http://service-b.staging/
```
