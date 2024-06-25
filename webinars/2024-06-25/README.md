# Introduction
These are notes for a webinar from 2024-06-25 on Envoy Gateway by me (Erica).

_Postman collection_
https://www.postman.com/e5r34t/workspace/webinars/collection/1269888-809916eb-13b0-4c84-b5d9-f0e337d461b9?action=share&creator=1269888 


# Taylor

## Create Taylor Namespace
```
kubectl create namespace taylor
```

## HTTPS with Self Signed Cert

### Create taylor.missberg.com Self Signed Cert

#### Create a root certificate and private key to sign certificates:
```
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=missberg.com' -keyout missberg.com.key -out missberg.com.crt
```

#### Create a certificate and a private key for taylor.missberg.com
```
openssl req -out taylor.missberg.com.csr -newkey rsa:2048 -nodes -keyout taylor.missberg.com.key -subj "/CN=taylor.missberg.com/O=example organization"
```
```
openssl x509 -req -days 365 -CA missberg.com.crt -CAkey missberg.com.key -set_serial 0 -in taylor.missberg.com.csr -out taylor.missberg.com.crt
```

#### Create secret in K8s
```
kubectl create secret tls taylor-cert -n taylor --key=taylor.missberg.com.key --cert=taylor.missberg.com.crt
```

## Create Taylor Gateway
```
kubectl apply -f taylor/gateway.yml
```
```
kubectl get gateway/taylor-gateway -n taylor
```
```
kubectl describe gateway/taylor-gateway -n taylor
```
```
kubectl get svc -n envoy-gateway-system -l gateway.envoyproxy.io/owning-gateway-namespace=taylor
```

## Create Taylor Blue & Green Services
```
kubectl apply -f taylor/blue.yml
```
```
kubectl apply -f taylor/green.yml
```

## Add a global ratelimit for the Gateway
```
kubectl apply -f taylor/global-rate-limit.yml
```

This will automatically apply to all routes on this gateway

## Taylor Blue-Green Release
Ok so now we want to have one route that demonstrates a blue-green release.
```
kubectl apply -f taylor/blue-green.yml
```

# Shared Gateway
I've created a shared gateway to be there to handle traffic from the outside to multiple namespaces.

```
kubectl get gateway/shared-gateway -n shared-gateway
```

I've created a couple of listeners.

## Add Taylor to Shared Gateway
Make sure we have the label so we can add routes to the shared gateway

```
kubectl label namespace taylor purpose=workloads
```
```
kubectl apply -f taylor/taylor-route.yml 
```

## Add Lucky Dip to Shared Gateway
```
kubectl apply -f lucky-dip/lucky-dip-route.yml
```

# Cert Manager with Let's Encrypt

Some stuff to note that I've done to set this up:
- I've created a DNS A record for gateway.demo.missberg.com for the shared gateway IP
- I have indeed set up the cert manager ahead of time, but I'm going to talk about how it works and how it is set up

## Install Cert Manager

```
helm repo add jetstack https://charts.jetstack.io
```
```
helm upgrade --install --create-namespace --namespace cert-manager --set installCRDs=true --set featureGates=ExperimentalGatewayAPISupport=true cert-manager jetstack/cert-manager
```
### Check installation

```
kubectl wait --for=condition=Available deployment -n cert-manager --all
```
```
kubectl get -n cert-manager deployment
```

## Overview of Setup

Useful links to understand what is happening in detail:

https://cert-manager.io/docs/usage/gateway/#inner-workings-diagram-for-developers 

https://cert-manager.io/docs/usage/certificate/#inner-workings-diagram-for-developers

### Step by step
- Install Cert Manager
- Create a ClusterIssuer
    - Verify the issuer is created `kubectl get clusterissuers.cert-manager.io --all-namespaces`
- Tell your Gateway to have certs mnanaged by the cert manager
- Create a listener on the Gateway that has a cert
- Check the Gateway `kubectl describe gateway/shared-gateway`
- When the cert manager notices that the Gateway needs a certificate, it will create the Certificate
    - Check it `kubectl get certificate --all-namespaces`
- The cert will be populated in a secret and the Certificate references the secret (same secret the Gateway listener is referencing)

# Lucky Dip
I promised you'll know how to always win on lucky dip...

Some general notes on the setup here:
- A namespace called `lucky-dip`
    - With the label `purpose=workloads` - so it can add routes to the shared-gateway
- Two services, you can find the files to build the images in the folder `go-services` under `lucky-dip`
    - Winner
    - Loser
- A route that splits traffic (similar to the Taylor blue-green route)
- A rate limiter, based on IP, because you only get so many chances


# General Helpful Resources

## Migrate from Ingress API to Gateway API
- https://github.com/kubernetes-sigs/ingress2gateway

## Kubernetes Gateway API
- https://gateway-api.sigs.k8s.io/#introduction
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
- https://cert-manager.io/v1.9-docs/configuration/acme/http01/#configuring-the-http-01-gateway-api-solver - Make sure this is turned on!
- https://cert-manager.io/docs/usage/gateway/#inner-workings-diagram-for-developers - how it actually works
- https://gateway.envoyproxy.io/v1.0.2/tasks/security/tls-cert-manager/
- https://cert-manager.io/docs/usage/gateway/

## Some local setup for GCP
- https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl
