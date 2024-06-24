# Taylor

## Create Taylor Namespace
kubectl create namespace -n taylor

## HTTPS with Self Signed Cert

### Create taylor.missberg.com Self Signed Cert

#### Create a root certificate and private key to sign certificates:
```
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=missberg.com' -keyout missberg.com.key -out missberg.com.crt

```

#### Create a certificate and a private key for taylor.missberg.com
```
openssl req -out taylor.missberg.com.csr -newkey rsa:2048 -nodes -keyout taylor.missberg.com.key -subj "/CN=taylor.missberg.com/O=example organization"
openssl x509 -req -days 365 -CA missberg.com.crt -CAkey missberg.com.key -set_serial 0 -in taylor.missberg.com.csr -out taylor.missberg.com.crt

```

#### Create secret in K8s
```
kubectl create secret tls taylor-cert -n taylor --key=taylor.missberg.com.key --cert=taylor.missberg.com.crt

```

## Create Taylor Gateway
kubectl apply -f taylor/gateway.yml
kubectl get gateway/taylor-gateway -n taylor
kubectl describe gateway/taylor-gateway -n taylor

## Create Taylor Blue & Green Services
kubectl apply -f taylor/blue.yml
kubectl apply -f taylor/green.yml

## Add a global ratelimit for the Gateway
kubectl apply -f taylor/global-rate-limit.yml
This will automatically apply to all routes on this gateway

## Taylor Blue-Green Release

# Shared Gateway

## Add Taylor to Shared Gateway
Make sure we have the label so we can add routes to the shared gateway

kubectl label namespace taylor purpose=workloads

kubectl apply -f taylor/taylor-route.yml 

## Add Lucky Dip to Shared Gateway
kubectl apply -f lucky-dip/lucky-dip-route.yml

# General Helpful Resources

## Kubernetes Gateway API
- https://gateway-api.sigs.k8s.io/api-types/gateway/
- https://gateway-api.sigs.k8s.io/api-types/gatewayclass/
- https://gateway-api.sigs.k8s.io/api-types/httproute/
- https://gateway-api.sigs.k8s.io/reference/spec/ 

## Traffic Management with Envoy Gateway
- https://gateway.envoyproxy.io/v1.0.2/tasks/traffic/http-traffic-splitting/
- https://gateway.envoyproxy.io/v1.0.2/tasks/traffic/http-urlrewrite/
- https://gateway.envoyproxy.io/v1.0.2/tasks/traffic/global-rate-limit/ 

## Cert Manager with Envoy Gateway
- https://gateway.envoyproxy.io/v1.0.2/tasks/security/tls-cert-manager/#monitoring-progress--troubleshooting


## Cert Manager
- https://gateway.envoyproxy.io/v1.0.2/tasks/security/tls-cert-manager/
- https://cert-manager.io/docs/usage/gateway/
