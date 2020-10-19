# istio-jwt
Istio Ingress Gateway with JWT token checking

# How to create it
* Spin up a new cluster
```shell
kind create cluster --name istio-jwt
yes | istioctl install
```

* Install Grafana and Prometheus to see metrics for the control plane

Each Istio component exposes an endpoint with metrics that are scraped by Prometheus.

```shell
cd istio # Git submodule "https://github.com/istio/istio.git"

kubectl apply -f ./samples/addons/prometheus.yaml
kubectl apply -f ./samples/addons/grafana.yaml
```

After the deployment has finished, you can get to the dashboard with:
```shell
istioctl dashboard grafana
```

* Install Kiali dashboard to visualize your service mesh
```
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.7/samples/addons/kiali.yaml
```

After the deployment has finished, you can get to the dashboard with:
```shell
istioctl dashboard kiali
```

* Start "httpbin" demo app
```shell
# cd istio
kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)

export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
export TCP_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="tcp")].nodePort}')
export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
```

* Expose httbin.example.com through Igress Gateway
```shell
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "httpbin.example.com"
EOF
```

* Set what URL path prefixes are allowed through an Ingress VirtualService
```shell
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "httpbin.example.com"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /status
    - uri:
        prefix: /delay
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
```

* Check you can reach to httpbin app
```shell
curl -s -I -H "Host: httpbin.example.com" "http://$INGRESS_HOST:$INGRESS_PORT/status/200"
```

* Add JWT Token Authentication and Authorization
This part uses an issued token prepared here: https://raw.githubusercontent.com/istio/istio/release-1.7/security/tools/jwt/samples/jwks.json
You can inspect the token with https://jwt.io/

```shell
kubectl apply -f - <<EOF
apiVersion: "security.istio.io/v1beta1"
kind: "RequestAuthentication"
metadata:
  name: "jwt-example"
  namespace: istio-system
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.7/security/tools/jwt/samples/jwks.json"
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: istio-system
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  action: ALLOW
  rules:
  - from:
    - source:
       requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
EOF
```

* Fetch the token and test your service without and with the token
```shell
TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.7/security/tools/jwt/samples/demo.jwt -s) && echo "$TOKEN" | cut -d '.' -f2 - | base64 --decode -

# this should work and you should get HTTP 200
curl -s -I -H "Host:httpbin.example.com" \
    -H "Authorization: Bearer $TOKEN" \
    "http://$INGRESS_HOST:$INGRESS_PORT/status/200"

# and this shouldn't work, you should get HTTP 403
curl -s -I -H "Host: httpbin.example.com" "http://$INGRESS_HOST:$INGRESS_PORT/status/200"
```

# Cleanup
```shell
kind delete cluster --name istio-jwt
```
