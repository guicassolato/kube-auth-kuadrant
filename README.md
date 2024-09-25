# Kubernetes Auth with Kuadrant

Demo tutorial of protecting a service application with Kubernetes Auth and Rate Limiting using [Kuadrant](https://kuadrant.io).

### Highlighted capabilities
- Kubernetes Auth ([TokenReview](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-review-v1/) and [SubjectAccessReview](https://kubernetes.io/docs/reference/kubernetes-api/authorization-resources/subject-access-review-v1/)) • powered by Kuadrant [`AuthPolicy`](https://docs.kuadrant.io/0.10.0/kuadrant-operator/doc/auth/) and [Authorino](https://docs.kuadrant.io/0.10.0/authorino/)
- Rate Limiting • powered by Kuadrant [`RateLimitPolicy`](https://docs.kuadrant.io/0.10.0/kuadrant-operator/doc/rate-limiting/) and [Limitador](https://docs.kuadrant.io/0.10.0/limitador/)
- Auto TLS certificate management • powered by Kuadrant [`TLSPolicy`](https://docs.kuadrant.io/0.10.0/kuadrant-operator/doc/tls/) and [cert-manager](https://cert-manager.io/)

### Stack
- [Kubernetes](https://kubernetes.io)
- [Gateway API](https://gateway-api.sigs.k8s.io/)
- [Istio](https://istio.io)
- [Kuadrant](https://kuadrant.io)
- [cert-manager](https://cert-manager.io/)
- [MetalLB](https://metallb.org/)

### Topology and request flow

![Architecture](architecture.png)

### Requirements

- [Docker](https://www.docker.com/) or [Podman](https://podman.io/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/introduction/)
- [Kind](https://kind.sigs.k8s.io/)
- [Helm](https://helm.sh/)
- [curl](https://curl.se)
- [yq](https://github.com/mikefarah/yq)

## Run the demo

### Deploy an application as usual

Create a cluster:

```sh
kind create cluster --name kuadrant-demo
```

Deploy the **News Agency API** application (`news-api`):

```sh
kubectl apply -f -<<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: news-api
  labels:
    app: news-api
spec:
  selector:
    matchLabels:
      app: news-api
  template:
    metadata:
      labels:
        app: news-api
    spec:
      containers:
      - name: news-api
        image: quay.io/kuadrant/authorino-examples:news-api
        imagePullPolicy: Always
        env:
        - name: PORT
          value: "3000"
        tty: true
        ports:
        - containerPort: 3000
  replicas: 1
---
apiVersion: v1
kind: Service
metadata:
  name: news-api
  labels:
    app: news-api
spec:
  selector:
    app: news-api
  ports:
  - name: http
    port: 3000
    protocol: TCP
EOF

kubectl wait deployment/news-api --for=condition=Available --timeout=5m
```

Expose and send requests to the application:

```sh
kubectl port-forward service/news-api 3000:3000 2>&1 >/dev/null &
```

```sh
curl http://localhost:3000/sports
# []
```

### Setup a gateway

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

### Enable security capabilities for the gateways

Install cert-manager:

```sh
helm repo add jetstack https://charts.jetstack.io --force-update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.15.3 \
  --set crds.enabled=true
```

Install Kuadrant:

```sh
helm repo add kuadrant https://kuadrant.io/helm-charts/ --force-update
helm install kuadrant-operator kuadrant/kuadrant-operator --devel
```

```sh
kubectl -n kuadrant-system apply -f - <<EOF
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant
spec: {}
EOF
```

Enable TLS on the gateway with automatic certificate management:

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
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - name: ingress-gateway-cert
        kind: Secret
    hostname: "*.io"
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: kuadrant-ca
spec:
  selfSigned: {}
---
apiVersion: kuadrant.io/v1alpha1
kind: TLSPolicy
metadata:
  name: ingress-gateway-tls
  namespace: istio-system
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: ingress-gateway
  issuerRef:
    group: cert-manager.io
    kind: ClusterIssuer
    name: kuadrant-ca
EOF
```

Try to consume the News Agency API protected by Kuadrant:

```sh
curl --resolve api.news.io:443:$INGRESS_IP https://api.news.io/sports
# curl: (60) SSL certificate problem: unable to get local issuer certificate
# More details here: https://curl.se/docs/sslcerts.html
#
# curl failed to verify the legitimacy of the server and therefore could not
# establish a secure connection to it. To learn more about this situation and
# how to fix it, please visit the web page mentioned above.
```

```sh
curl --resolve api.news.io:443:$INGRESS_IP https://api.news.io/sports --insecure
# []
```

### Add authentication and authorization to the application

Create an AuthPolicy:

```sh
kubectl apply -f -<<EOF
apiVersion: kuadrant.io/v1beta2
kind: AuthPolicy
metadata:
  name: news-api-auth
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: news-api
  rules:
    authentication:
      k8s-tokens:
        kubernetesTokenReview:
          audiences:
          - https://kubernetes.default.svc.cluster.local
    authorization:
      k8s-rbac:
        kubernetesSubjectAccessReview:
          user:
            selector: auth.identity.user.username
          resourceAttributes:
            group:
              value: news-api.io
            verb:
              selector: request.method.@case:lower
            resource:
              selector: request.path.@extract:{"sep":"/","pos":1}
            namespace:
              value: default
EOF
```

Try the News Agency API with auth enabled:

```sh
curl -k --resolve api.news.io:443:$INGRESS_IP https://api.news.io/sports -i
# HTTP/2 401
# www-authenticate: Bearer realm="k8s-tokens"
# x-ext-auth-reason: credential not found
```

Create a ServiceAccount to represent a user called Alice inside the cluster:

```sh
kubectl apply -f -<<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: alice
EOF
```

Consume the News Agency API with a valid access token:

```sh
curl -k --resolve api.news.io:443:$INGRESS_IP -H "Authorization: Bearer $(kubectl create token alice)" https://api.news.io/sports -i
# HTTP/2 403
```

Define roles:

```sh
kubectl apply -f -<<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: publisher
rules:
- apiGroups: ["news-api.io"]
  resources:
  - sports
  - economy
  verbs:
  - post
  - get
  - delete
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: sports-reader
rules:
- apiGroups: ["news-api.io"]
  resources: ["sports"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: economy-reader
rules:
- apiGroups: ["news-api.io"]
  resources: ["economy"]
  verbs: ["get"]
EOF
```

Add permissions for Alice:

```sh
kubectl apply -f -<<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sports-readers
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: sports-reader
subjects:
- kind: ServiceAccount
  name: alice
EOF
```

Consume the application again now with permissions:

```sh
curl -k --resolve api.news.io:443:$INGRESS_IP -H "Authorization: Bearer $(kubectl create token alice)" https://api.news.io/sports
# []
```

Try a forbidden endpoint:

```sh
curl -k --resolve api.news.io:443:$INGRESS_IP -H "Authorization: Bearer $(kubectl create token alice)" https://api.news.io/economy -i
# HTTP/2 403
```

Create another user, called Niko, with permission to publish news articles:

```sh
kubectl apply -f -<<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: niko
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: publishers
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: publisher
subjects:
- kind: ServiceAccount
  name: niko
EOF
```

Create a news article as Niko:

```sh
curl -k --resolve api.news.io:443:$INGRESS_IP -H "Authorization: Bearer $(kubectl create token niko)" \
  -X POST -d '{"title": "Italy’s Luna Rossa showed two skippers are better than one on the America’s Cup yachts", "body": "BARCELONA, Spain (AP) — Before the last America’s Cup, Italy’s sailing team had what helmsman Francesco Bruni called a “crazy” idea: run the boat with two skippers, each taking turns steering as the foiling yacht crisscrossed the race course.\nWhile the other crews lost valuable time as their sole skipper scampered back and forth with each tack or jibe, Bruni and Jimmy Spithill stayed put, each manning their own helm on their side of the boat.\nIt turned out to be a stroke of genius. Source: https://apnews.com/article/americas-cup-sailing-italy-luna-rossa-spithill-d99078e7047bda9d9416c92e333a345d"}' \
  https://api.news.io/sports
# {"id": …}
```

Read news articles as Niko:

```sh
curl -k --resolve api.news.io:443:$INGRESS_IP -H "Authorization: Bearer $(kubectl create token niko)" https://api.news.io/sports
# [{"id": …}]
```

```sh
curl -k --resolve api.news.io:443:$INGRESS_IP -H "Authorization: Bearer $(kubectl create token niko)" https://api.news.io/economy
# []
```

### Add rate limiting to the application

Create a RateLimitPolicy:

```sh
kubectl apply -f - <<EOF
apiVersion: kuadrant.io/v1beta2
kind: RateLimitPolicy
metadata:
  name: news-api-rate-limit
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: news-api
  limits:
    global:
      rates:
      - limit: 2
        duration: 10
        unit: second
      counters:
      - request.path
EOF
```

Try to consume the News Agency API with limits enforced:

```sh
for i in {1..10}; do curl -k --resolve api.news.io:443:$INGRESS_IP -H "Authorization: Bearer $(kubectl create token niko)" https://api.news.io/sports && sleep 1; done
```

```sh
for i in {1..10}; do curl -k --resolve api.news.io:443:$INGRESS_IP -H "Authorization: Bearer $(kubectl create token niko)" https://api.news.io/economy && sleep 1; done
```

### Cleanup

Delete the cluster:

```sh
kind delete cluster --name kuadrant-demo
```
