# bookinfo-knative

## Minikube example setup
Follow the instructions bellow:

Install istio.

```
$ cat ~/istio-minimal-operator.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      proxy:
        autoInject: enabled
      useMCP: false
      # The third-party-jwt is not enabled on all k8s.
      # See: https://istio.io/docs/ops/best-practices/security/#configure-third-party-service-account-tokens
      jwtPolicy: first-party-jwt

  addonComponents:
    pilot:
      enabled: true

  components:
    ingressGateways:
      - name: istio-ingressgateway
        enabled: true

$istioctl manifest --set values.gateways.istio-ingressgateway.runAsRoot=true install -f istio-minimal-operator.yaml
```

In a separate terminal run:

```
minikube tunnel
```

Install Knative Serving:

```
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.22.0/serving-crds.yaml
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.22.0/serving-core.yaml
kubectl apply --filename https://github.com/knative/net-istio/releases/download/v0.22.0/istio.yaml
kubectl apply --filename https://github.com/knative/net-istio/releases/download/v0.22.0/net-istio.yaml
```

Install bookinfo sample app (emptyDir is not support on Knative):

```
cat <<-EOF | kubectl apply -f -
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-details
  labels:
    account: details
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: details-v1
  namespace: default
  labels:
    app: details
    version: v1
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "1"
    spec:
      serviceAccountName: bookinfo-details
      containers:
        - name: details
          image: docker.io/istio/examples-bookinfo-details-v1:1.16.2
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 9080
          securityContext:
            runAsUser: 1000
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-ratings
  labels:
    account: ratings
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: ratings-v1
  namespace: default
  labels:
    app: ratings
    version: v1
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "1"
    spec:
      serviceAccountName: bookinfo-details
      containers:
        - name: ratings
          image: docker.io/istio/examples-bookinfo-ratings-v1:1.16.2
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 9080
          securityContext:
            runAsUser: 1000
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-reviews
  labels:
    account: reviews
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: reviews-v1
  namespace: default
  labels:
    app: reviews
    version: v1
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "1"
    spec:
      serviceAccountName: bookinfo-reviews
      containers:
        - name: reviews
          image:  docker.io/istio/examples-bookinfo-reviews-v1:1.16.2
          imagePullPolicy: IfNotPresent
          env:
            - name: LOG_DIR
              value: "/tmp/logs"
          ports:
            - containerPort: 9080
          securityContext:
            runAsUser: 1000
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: reviews-v2
  namespace: default
  labels:
    app: reviews
    version: v2
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "1"
    spec:
      serviceAccountName: bookinfo-reviews
      containers:
        - name: reviews
          image:  docker.io/istio/examples-bookinfo-reviews-v2:1.16.2
          imagePullPolicy: IfNotPresent
          env:
            - name: LOG_DIR
              value: "/tmp/logs"
          ports:
            - containerPort: 9080
          securityContext:
            runAsUser: 1000
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: reviews-v3
  namespace: default
  labels:
    app: reviews
    version: v3
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "1"
    spec:
      serviceAccountName: bookinfo-reviews
      containers:
        - name: reviews
          image:  docker.io/istio/examples-bookinfo-reviews-v3:1.16.2
          imagePullPolicy: IfNotPresent
          env:
            - name: LOG_DIR
              value: "/tmp/logs"
          ports:
            - containerPort: 9080
          securityContext:
            runAsUser: 1000
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-productpage
  labels:
    account: productpage
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: productpage-v1
  namespace: default
  labels:
    app: productpage
    version: v1
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "1"
    spec:
      serviceAccountName: bookinfo-productpage
      containers:
        - name: productpage
          image:  docker.io/istio/examples-bookinfo-productpage-v1:1.16.2
          imagePullPolicy: IfNotPresent
          env:
            - name: LOG_DIR
              value: "/tmp/logs"
          ports:
            - containerPort: 9080
          securityContext:
            runAsUser: 1000    
EOF
```

Test your setup:

```
$ curl -H "Host: details-v1.default.example.com" http://192.168.39.171:31640/details/0 | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   178  100   178    0     0  16181      0 --:--:-- --:--:-- --:--:-- 16181
{
  "id": 0,
  "author": "William Shakespeare",
  "year": 1595,
  "type": "paperback",
  "pages": 200,
  "publisher": "PublisherA",
  "language": "English",
  "ISBN-10": "1234567890",
  "ISBN-13": "123-1234567890"
}

$ curl -H "Host: reviews-v1.default.example.com" http://192.168.39.171:31640/reviews/0 | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   295  100   295    0     0    438      0 --:--:-- --:--:-- --:--:--   438
{
  "id": "0",
  "reviews": [
    {
      "reviewer": "Reviewer1",
      "text": "An extremely entertaining play by Shakespeare. The slapstick humour is refreshing!"
    },
    {
      "reviewer": "Reviewer2",
      "text": "Absolutely fun and entertaining. The play lacks thematic depth when compared to other plays by Shakespeare."
    }
  ]
}
```
