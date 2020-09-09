# apim-dapr

> NOte, this is still work in progress

## Prerequisite 

* [Azure account](https://azure.microsoft.com/en-us/free/)
* [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
* [Kubernetes with Dapr](https://github.com/mchmarny/dapr-demos/tree/master/setup)

## Setup 

Assumes your default Azure resource group and location are already defined. If not, run:

```shell
az account set --subscription <id or name>
az configure --defaults location=<preferred location> group=<preferred resource group>
```

## Azure API Management Configuration 

> Note, the name of your API Management service instance name (`daprapimdemo`) has to be globally unique!

Create APIM service instance:

```shell
az apim create --name daprapimdemo \
               --sku-name Premium \
               --enable-client-certificate \
               --publisher-email mchmarny@microsoft.com \
               --publisher-name "OCTO Azure"
```

> Note, depending on your configuration, this operation may take 10+ min

Update the [api.yaml](./api.yaml) with the `gatewayUrl`

```yaml
servers:
  - url: http://daprapimdemo.azure-api.net
  - url: https://daprapimdemo.azure-api.net
```

Make sure you save the [api.yaml](./api.yaml) when finished and now import it into the previously created APIM service instance:

```shell
az apim api import --path /echo \
                   --service-name daprapimdemo \
                   --display-name echo-service \
                   --protocols http \
                   --subscription-required false \
                   --specification-path api.yaml \
                   --specification-format OpenApi
```

> How much what's in the `api.yaml` is really required here? 

Get API Token and the newly created API ID

```shell
export AZ_API_TOKEN=$(az account get-access-token --resource=https://management.azure.com --query accessToken --output tsv)
export AZ_API_ID=$(az apim api list --service-name daprapimdemo --query "[?contains(displayName, 'echo-service')].id" --output tsv)
```

Now apply the Dapr backend policy: 

```shell
curl -v -X PUT -d @./policy.json \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com${AZ_API_ID}/operations/post-echo/policies/policy?api-version=2019-12-01"
```

> Note, this succeeds (status code 201) and the valid, created policy is returned by the API but when you go to Azure portal it displays an error that the policies could not be loaded, come again later. Is it just a readiness check issue? The policy is eventually created and visible in portal.

Create Gateway 

```shell
export AZ_API_GATEWAY=$(echo $AZ_API_ID | rev | cut -d'/' -f3- | rev)
```

> Yes, this is all kinds of voodoo to get make this reproducible without asking you to cut and paste parts of the ID.

```shell
curl -v -X PUT -d '{"properties": {"description": "Dapr Gateway","locationData": {"name": "Virtual"}}}' \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com${AZ_API_GATEWAY}/gateways/gw1?api-version=2019-12-01"
```

Map Gateway to API

```shell
export AZ_API_OP_ID=$(echo $AZ_API_ID | cut -d'/' -f11-)
```

```shell
curl -v -X PUT -d '{ "properties": { "provisioningState": "created" } }' \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com${AZ_API_GATEWAY}/gateways/gw1/apis/${AZ_API_OP_ID}?api-version=2019-12-01"
```

## Dapr Service Deployment 

Deploy a Dapr service and watch it until it's ready:

```shell
kubectl apply -f ./service.yaml
watch kubectl get pods -l app=echo-service
```

> Service is ready when its status is `Running` and the ready column is `2/2` (Dapr and our echo service both started)

```shell
NAME                            READY   STATUS    RESTARTS   AGE
echo-service-77d6f5b5bb-crc5q   2/2     Running   0          97s
```

> Note, you may have to restart the gateway if you have deployed/updated your service AFTER the gateway

```shell
kubectl rollout restart deployment/daprdemo-gateway
kubectl rollout status deployment/daprdemo-gateway
```

## Deploy APIM Gateway 

> Can't find any way to create a gateway programmatically (neither in Az CLI, or REST API)

Create a secret 

```shell
kubectl create secret generic gw1-token --type=Opaque --from-literal=value="GatewayKey ..."  
```

Augment the deployment template metadata with Dapr annotations:

> Note, this step may be not required (or at least not for all the annotations) when automated 

```yaml
annotations:
    dapr.io/enabled: "true"
    dapr.io/app-id: "daprdemo-gateway"
    dapr.io/config: "tracing"
    dapr.io/log-as-json: "true"
    dapr.io/log-level: "debug"
```

Deploy gateway

```shell
kubectl apply -f ./gateway.yaml
watch kubectl get pods -l app=gw1
```

Wait for both instances (required for rolling upgrades) are status `STATUS` and container is ready `1/1`.

```shell
NAME                           READY   STATUS    RESTARTS   AGE
gw1-7896ddc989-m4lmm   2/2     Running   0          26s
gw1-7896ddc989-ts8cm   2/2     Running   0          26s
```

## Test

```shell
export GATEWAY_IP=$(kubectl get svc gw1 -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

```shell
curl -v -X POST -d '{"message": "hello"}' \
     -H "Content-Type: application/json" \
     "http://${GATEWAY_IP}/echo"
```

## Debug 


### Backing Service 

Check if the backing service (``) is working. First forward the service port:

```shell
kubectl port-forward pod/echo-service-944d8684b-nbzxz 3500:3500
```

Call the service using the forwarded port 

```shell
curl -v -X POST -d '{"message": "hello"}' \
     -H "Content-Type: application/json" \
     "http://localhost:3500/v1.0/invoke/echo-service/method/echo"
```

If service is running correctly you will see status 200 (`HTTP/1.1 200 OK`) and the sent message returned `{"message": "hello"}`

### Gateway 

When you see `404` errors from invoking the gateway check the gateway logs 

```shell
kubectl logs -l app=gw1 -c gw1 -f
```

If you see entries about inability to `match incoming request to an operation` check your policy

```shell
[Info] 2020-09-9T01:48:03.746, time: 09/09/2020 13:48:03, apiId: 9ab8d6aa9fe64a02a860bdf91dd0ca55, apiRevision: 1, isCurrentApiRevision: 1, clientIp: 67.189.125.3, url: http://20.51.70.83/echo, method: POST, responseCode: 404, backendResponseCode: 0, requestSize: 0, responseSize: 130, clientProtocol: HTTP/1.1, totalTime: 208, backendTime: 0, clientTime: 1, cacheTime: 0, cacheType: 0, errorSource: configuration, errorReason: OperationNotFound, errorMessage: Unable to match incoming request to an operation., errorSection: backend, errorElapsed: 172, tags: 546, tokensTime: 0, invalidOpKey: 0
```


## Cleanup 

```shell
az apim delete --name daprapimdemo --no-wait --yes
```