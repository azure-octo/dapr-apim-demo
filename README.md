# apim-dapr

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

> Note, this operation will take a while (~10 min depending on your configuration)

The resulting JSON is little long, the two fields you will need to capture are: the gateway URL, and the public IP:

```json
{
  "gatewayUrl": "https://daprapimdemo.azure-api.net",
  "publicIpAddresses": [ "52.250.71.202" ]
}
```

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
     "https://management.azure.com${AZ_API_ID}/policies/policy?api-version=2019-12-01"
```

> Note, this succeeds (status code 201) and the valid, created policy is returned by the API but when you go to Azure portal it displays an error that the policies could not be loaded, come again later. Is it just a readiness check issue? The policy is eventually created and visible in portal.

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

If you are applying update after the gateway was deployed make sure to restart the gateway 

```shell
kubectl rollout restart deployment/daprdemo-gateway
kubectl rollout status deployment/daprdemo-gateway
```

## Deploy APIM Gateway 

> Can't find any way to create a gateway programmatically (neither in Az CLI, or REST API)

Create a secret 

```shell
kubectl create secret generic daprdemo-token --type=Opaque --from-literal=value="GatewayKey ..."  
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
watch kubectl get pods -l app=daprdemo-gateway
```

Wait for both instances (required for rolling upgrades) are status `STATUS` and container is ready `1/1`.

```shell
NAME                                READY   STATUS    RESTARTS   AGE
daprdemo-gateway-7896ddc989-m4lmm   2/2     Running   0          26s
daprdemo-gateway-7896ddc989-ts8cm   2/2     Running   0          26s
```

## Test

```shell
export GATEWAY_IP=$(kubectl get svc daprdemo-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

```shell
curl -v -X POST -d '{"message": "hello"}' \
     -H "Content-Type: application/json" \
     "http://${GATEWAY_IP}/echo"
```

## Cleanup 

```shell
az apim delete --name daprapimdemo --no-wait --yes
```