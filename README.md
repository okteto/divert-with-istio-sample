# Sample for Divert with Istio
This repo is a sample about using [Okteto Divert](https://www.okteto.com/docs/reference/okteto-manifest/#divert) with istio.
The following diagram explains the architecture of the application used in this sample:

![sample-app](images/sample-app.png)

### Install Okteto with support for Istio/Divert

Deploy an Okteto instance. Once the Okteto instance is operational, add the following Helm configuration and upgrade it:

```yaml
okteto-nginx:
  enabled: false

virtualServices:
  enabled: true

namespace:
  labels:
    istio-injection: enabled
```

- Disable `okteto-nginx` because we will enable istio ingress mode to expose user applications
- Enable `virtualServices` to display virtual service endpoints in the Okteto UI
- Inject the label `istio-injection: enabled` on every namespace manage by Okteto. This label instructs Istio to add Istio sidecards on every pod to ensure pod traffic goes though the Istio Mesh

### Cluster Admin tasks

The cluster administrator must run the following commands to install Istio in the Okteto instance.


First, download the istio chart (last tested version is 1.25.2):

```
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

Export the `OKTETO_DOMAIN` envvar with the `subdomain` Helm value of your Okteto instance. For example:

```
export OKTETO_DOMAIN=okteto.example.com
```

Then, install the following Istio components:

```
helm install --namespace istio-system --create-namespace istio-base istio/base --wait
kubectl create namespace istio-ingress
kubectl label namespace istio-ingress istio-injection=enabled
helm install --namespace istio-system istiod istio/istiod --values istio/istiod-helm-values.yaml --wait
helm install --namespace istio-ingress istio-ingress istio/gateway --values istio/istio-ingress-helm-values.yaml --wait
envsubst < istio/istio-ingress-config.yaml | kubectl apply -n istio-ingress -f -
```

### Develop tasks

Each developer must run the following commands to deploy their development environment

### Configure Okteto CLI and kubectl credentials

```
okteto ctx use https://okteto.${OKTETO_DOMAIN}
okteto kubeconfig
```

### Deploy services in staging

Then, run `okteto deploy -n staging -f okteto-staging.yml` to deploy the application in the `staging` namespace.
This command will create the `staging` namespace if it doesn't exist.

### Test services

Run the command `okteto test -f okteto-staging.yml`:

```
 i  Executing test container 'endpoints'
 i  Running 'curl -s -k https://service-a-staging.${OKTETO_DOMAIN}'
this is service-a
 ✓  Command 'curl -s -k https://service-a-staging.${OKTETO_DOMAIN}' successfully executed
 i  Running 'curl -s -k https://service-b-staging.${OKTETO_DOMAIN}'
this is staging/service-b
 ✓  Command 'curl -s -k https://service-b-staging.${OKTETO_DOMAIN}' successfully executed
 i  Running 'curl -s -k https://service-c-staging.${OKTETO_DOMAIN}'
this is staging/service-c
 ✓  Command 'curl -s -k https://service-c-staging.${OKTETO_DOMAIN}' successfully executed
 i  Running 'curl -s -k https://service-a-staging.${OKTETO_DOMAIN}/call-b'
this is staging/service-b
 ✓  Command 'curl -s -k https://service-a-staging.${OKTETO_DOMAIN}/call-b' successfully executed
 i  Running 'curl -s -k https://service-a-staging.${OKTETO_DOMAIN}/call-c'
this is staging/service-c
 ✓  Command 'curl -s -k https://service-a-staging.${OKTETO_DOMAIN}/call-c' successfully executed
 ✓  Test container 'endpoints' passed
```

- The first command returns service-a in staging via its own endpoint.
- The second command returns service-b in staging via its own endpoint.
- The third command returns service-c in staging via its own endpoint.
- The fourth command returns service-b in staging via the [service-a "/call-b" path](/k8s/service-a-nginx.yaml#L22).
- The fifth command returns service-c in staging via the [service-a "/call-c" path](/k8s/service-a-nginx.yaml#L28).

### Divert service-b in your personal namespace

Run `okteto deploy`. This commands will deploy service-b in your personal namespace, and divert traffic from service-b in the `staging` namespace based on the `baggage: okteto-divert header`. It also clones the service-a virtual service host in your personal namespace to perform the header `baggage: okteto-divert header` injection.

### Test services

Run the command `okteto test`:

```
 i  Executing test container 'endpoints'
 i  Running 'curl -s -k https://service-a-${OKTETO_NAMESPACE}.${OKTETO_DOMAIN}'
this is service-a
 ✓  Command 'curl -s -k https://service-a-${OKTETO_NAMESPACE}.${OKTETO_DOMAIN}' successfully executed
 i  Running 'curl -s -k https://service-b-${OKTETO_NAMESPACE}.${OKTETO_DOMAIN}'
this is okteto-admin/service-b
 ✓  Command 'curl -s -k https://service-b-${OKTETO_NAMESPACE}.${OKTETO_DOMAIN}' successfully executed
 i  Running 'curl -s -k https://service-c-${OKTETO_NAMESPACE}.${OKTETO_DOMAIN}'
 ✓  Command 'curl -s -k https://service-c-${OKTETO_NAMESPACE}.${OKTETO_DOMAIN}' successfully executed
 i  Running 'curl -s -k https://service-a-${OKTETO_NAMESPACE}.${OKTETO_DOMAIN}/call-b'
this is okteto-admin/service-b
 ✓  Command 'curl -s -k https://service-a-${OKTETO_NAMESPACE}.${OKTETO_DOMAIN}/call-b' successfully executed
 i  Running 'curl -s -k https://service-a-${OKTETO_NAMESPACE}.${OKTETO_DOMAIN}/call-c'
this is staging/service-c
 ✓  Command 'curl -s -k https://service-a-${OKTETO_NAMESPACE}.${OKTETO_DOMAIN}/call-c' successfully executed
 i  Running 'curl -s -k https://service-a-staging.${OKTETO_DOMAIN}/call-b'
this is staging/service-b
 ✓  Command 'curl -s -k https://service-a-staging.${OKTETO_DOMAIN}/call-b' successfully executed
 i  Running 'curl -s -k https://service-a-staging.${OKTETO_DOMAIN}/call-c'
this is staging/service-c
 ✓  Command 'curl -s -k https://service-a-staging.${OKTETO_DOMAIN}/call-c' successfully executed
 ✓  Test container 'endpoints' passed
```

- The first command returns service-a in staging via your personal service-a endpoint.
- The second command returns service-b in your personal namespace via your personal service-b endpoint.
- The third command returns service-c in your personal namespace via your personal service-c endpoint.
- The fourth command returns service-b in your personal namespace via your personal service-a "/call-b" path (diverted!).
- The fifth command returns service-c in your personal namespace via your personal service-a "/call-c" path.
- The sixth command returns service-b in staging via the staging service-a "/call-b" path.
- The seventh command returns service-c in staging via the staging service-a "/call-c" path.
