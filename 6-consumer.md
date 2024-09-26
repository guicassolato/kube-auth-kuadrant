# [Kubernetes Auth with Kuadrant](README.md)

## Run the steps

#### ❶ [Deploy an application as usual](1-deploy.md) ☑️
#### ❷ [Setup a gateway](2-gateway.md) ☑️
#### ❸ [Enable security capabilities for the gateways](3-kuadrant.md) ☑️
#### ❹ [Add authentication and authorization to the application](4-auth.md) ☑️
#### ❺ [Add rate limiting to the application](5-rate-limit.md)
#### ❻ Deploy the consumer app

Deploy an application that tries to consume the News Agency API within the cluster at every 3 seconds:

```sh
kubectl create namespace consumer

kubectl apply -n consumer -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-consumer
---
apiVersion: v1
kind: Pod
metadata:
  name: app-consumer
spec:
  containers:
  - name: api-consumer
    image: quay.io/kuadrant/authorino-examples:api-consumer
    imagePullPolicy: Always
    args:
    - ./run
    - --endpoint=https://api.news.io/economy
    - --token-path=/var/run/secrets/tokens/access-token
    - --resolve=api.news.io:443:$INGRESS_IP
    - --insecure
    - --silent
    - --output=/dev/null
    - --write-out=%{http_code}
    - --interval=3
    volumeMounts:
    - name: access-token
      mountPath: /var/run/secrets/tokens
  serviceAccountName: app-consumer
  volumes:
  - name: access-token
    projected:
      sources:
      - serviceAccountToken:
          path: access-token
          expirationSeconds: 7200
EOF
```

Grant permission to the application to read the category of news articles:

```sh
kubectl apply -f -<<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: economy-readers
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: economy-reader
subjects:
- kind: ServiceAccount
  name: app-consumer
  namespace: consumer
EOF
```

Check the logs of the consumer app:

```sh
kubectl logs -f app-consumer -n consumer
# Sending...
# 200
# 200
# 429
# 429
# 200
```

<br/>

#### [Cleanup](README.md#cleanup)
