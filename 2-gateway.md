# [Kubernetes Auth with Kuadrant](README.md)

## Run the steps

#### ❶ [Deploy an application as usual](1-deploy.md) ☑️
#### ❷ Setup a gateway

Install Kubernetes Gateway API:

```sh
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml
```

Install the Istio Gateway Controller:

```sh
curl -sL https://istio.io/downloadIstio | ISTIO_VERSION=1.23.2 sh -
istio-1.23.2/bin/istioctl operator init --operatorNamespace istio-system

kubectl apply -f -<<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: istiocontrolplane
spec:
  profile: default
  namespace: istio-system
  components:
    base:
      enabled: true
    cni:
      enabled: false
    egressGateways:
      - enabled: false
        name: istio-egressgateway
    ingressGateways:
      - enabled: false
        name: istio-ingressgateway
    pilot:
      enabled: true
      k8s:
        resources:
          requests:
            cpu: "0"
  values:
    pilot:
      autoscaleEnabled: false
    global:
      istioNamespace: istio-system
EOF
```

Install MetalLB:

```sh
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
kubectl wait -n metallb-system deployment/controller --for=condition=Available --timeout=300s
kubectl wait -n metallb-system pod --selector=app=metallb --for=condition=ready --timeout=60s
curl -s https://raw.githubusercontent.com/Kuadrant/kuadrant-operator/refs/tags/v0.10.0/utils/docker-network-ipaddresspool.sh | bash -s -- kind yq 1 | kubectl apply -n metallb-system -f -
```

Deploy an ingress gateway:

```sh
kubectl apply -f -<<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ingress-gateway
  namespace: istio-system
spec:
  gatewayClassName: istio
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: "*.io"
    allowedRoutes:
      namespaces:
        from: All
EOF
```

Connect the News Agency API through the gateway:

```sh
kubectl apply -f -<<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: news-api
spec:
  parentRefs:
  - name: ingress-gateway
    namespace: istio-system
  hostnames:
  - api.news.io
  rules:
  - matches:
    - method: POST
    - method: GET
    - method: DELETE
    backendRefs:
    - name: news-api
      port: 3000
EOF
```

Consume the News Agency API through the gateway:

```sh
export INGRESS_IP=$(kubectl get gateway/ingress-gateway -n istio-system -o jsonpath='{.status.addresses[0].value}')
```

```sh
curl --resolve api.news.io:80:$INGRESS_IP http://api.news.io/sports
# []
```

#### ❸ [Enable security capabilities for the gateways](3-kuadrant.md)
#### ❹ [Add authentication and authorization to the application](4-auth.md)
#### ❺ [Add rate limiting to the application](5-rate-limit.md)
#### ❻ [Deploy the consumer app](6-consumer.md)

<br/>

#### [Cleanup](README.md#cleanup)
