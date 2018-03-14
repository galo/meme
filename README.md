# Managing your Micro Services policies with Istio Service Mesh

This document goes over the steps to deploy a Meme app on K8s with Istio.

## Setup clusters

Select the clusters you want to build. In this case we will deploy on 3 clusters and setup Istio following one of the [guides](https://istio.io/docs/setup/)

## Deploy Service

```shell
kubectl apply -f secret.yaml
kubectl apply -f deployment.yaml
```

If injection extension are not configured in the cluster, you can manually inject the proxy

```shell
kubectl  apply -f <(istioctl kube-inject -f deployment.yaml)
```

## Validate and get logs

```shell
kubectl get pods
```

```shell
kubectl describe pod <POD_ID>
```

```shell
kubectl get logs <POD_ID>
```

## Test the service

```shell
kubectl describe ingress
```

You will get two ingresses. We can start using the US one, so set the GATEWAY accordingly

```shell
GATEWAY=$(kubectl describe ingress | grep Address | awk '{print $2}')
```

Validate the meme service is running. You should get a response indicating the version and host name.

```shell
curl -sX GET http://{$GATEWAY}/meme/v1/health
```

## Access Grafana

```shell
kubectl port-forward -n istio-system $(kubectl get pod -l app=grafana -n istio-system -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
```

Then point your web browser to (http://localhost:3000/dashboard/db/istio-dashboard)

## Access the Meme APIs

API are protected by a token, in our case we will use a simple token, 

```json
{
  "sub": "galo@acme.com"
}
```

that we encode in base64, and will send as a special header _sec-istio-auth-userinfo_

```txt
ew0KICAic3ViIjogImdhbG9AYWNtZS5jb20iDQp9
```

Create a meme

```shell
curl -X POST \
  http://{$GATEWAY}/meme/v1/memes?text=hello \
  -H 'sec-istio-auth-userinfo:ew0KICAic3ViIjogImdhbG9AYWNtZS5jb20iDQp9'
```

Get memes for my user

```shell
curl -X GET http://{$GATEWAY}/meme/v1/memes  -H 'sec-istio-auth-userinfo: ew0KICAic3ViIjogImdhbG9AYWNtZS5jb20iDQp9'
```

Get a specific meme

```shell
curl -X GET http://{$GATEWAY}//meme/v1/memes/cbbcdd4a-c7f4-4d60-bedf-8a3c595e66fe  -H 'sec-istio-auth-userinfo: ew0KICAic3ViIjogImdhbG9AYWNtZS5jb20iDQp9'
```

## Set Policies

### External service access policy

```shell
curl -sX GET http://{$GATEWAY}/meme/v1/now
```

Opps. The service's _now_ endpoint (/meme/v1/now) will attempt to access another external service at now.httpbin.org, but it is currently blocnot allowed.

For the _now_ (/meme/v1/now) endpoint to work you need to create an [egress rule](https://istio.io/docs/tasks/traffic-management/egress.html) :

```shell
kubectl create -f rules/httpbin_egress_rule.yaml
```

Validate the service can access external services, such as S3, etc; the /now endpoint will make a call to httpbin.org.

```shell
curl -sX GET http://{$GATEWAY}/meme/v1/now
```

## Traffic Spliting

We deploy a new version of the imager service and will, and will configure traffic spliting

```shell
kubectl apply -f imager2_deployment.yaml
```

We can access the imager service annotate endpoint to get a render. We have exposed this enpoint in out ingress just for demostrating, usually this service will not be exposed outside the mesh.

We can set a rule that sends 90% of teh traffic to the old version, and 10% to the new version.

```shell
kubectl apply -f rules/canary_rule.yaml
```

## Auth

Since we want to have only protected access to the application api meme, we need to add an Istio mixer rule that will allow only authorized users to access the API.

Configure JWT ([JSON Web Token](https://jwt.io/introduction/)) auth in the mixer

Enable access to the token endpoint

```shell
kubectl create -f rules/apigee_egress_rule.yaml
```

```shell
kubectl create -f rules/auth_rule.yaml
```

Configure denier rule for non authorized access

```shell
kubectl create -f build/ambarli/rules/only_authorized_rule.yaml
```

### Test the services

Providing an invalid token will fail

```shell
curl -vvv http://{$GATEWAY}/meme/v1/health -H 'Authorization: Bearer 2334'
```

Get a valid token by following the [auth guide](https://github.azc.ext.hp.com/gdrs/meme/blob/master/docs/auth.md).

```shell
curl -vvv http://{$GATEWAY}/meme/v1/health -H 'Authorization: Bearer <valid-jwt>'
```

If the token expired, the iss or audience are not matching, you will get an error in the response payload such as JWT_EXPIRED.

### Deleting deployments and rules

```shell
kubectl delete -f rules/canary_rule.yaml
kubectl delete -f rules/apigee_egress_rule.yaml
kubectl delete -f rules/httpbin_egress_rule.yaml
kubectl delete -f rules/auth_rule.yaml
kubectl delete -f rules/only_authorized_rule.yaml
kubectl delete -f deployment.yaml
kubectl delete -f imager2_deployment.yaml
```