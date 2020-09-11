# Dapr & Azure API Management Integration Demo

This repo demonstrates how to expose Dapr API and invoke a service on Kubernetes using [Azure API Management](https://azure.microsoft.com/en-us/services/api-management/) (APIM) self-hosted gateway.

## Prerequisite 

* [Azure account](https://azure.microsoft.com/en-us/free/)
* [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
* [Kubernetes cluster with Dapr](https://github.com/dapr/docs/blob/v0.9.0/getting-started/environment-setup.md#installing-dapr-on-a-kubernetes-cluster)
* [Helm](https://helm.sh/docs/intro/install/) to install Redis

## Setup 

To make this demo easier to reproduce, start by exporting the name of the Azure API Management (APIM) service we will create.

> Note, the name of your API Management service instance name has to be globally unique!

```shell
export APIM_SERVICE_NAME="dapr-apim-demo"
```

In addition to the above name, export also the Azure [Subscription ID](https://docs.bitnami.com/azure/faq/administration/find-subscription-id/) and [Resource Group](https://docs.bitnami.com/azure/faq/administration/find-deployment-resourcegroup-id/) where you would like to create these APIM service.

```shell
export AZ_SUBSCRIPTION_ID="your-subscription-id"
export AZ_RESOURCE_GROUP="your-resource-group"
```

## Azure API Management Configuration 

In this section we will create all the Azure resources. First, create and configure the Azure API Management service.

### Service

Create APIM service instance:

> The `publisher-email` and `publisher-name` below are required to receive system notifications e-mails.

```shell
az apim create --name $APIM_SERVICE_NAME \
               --subscription $AZ_SUBSCRIPTION_ID \
               --resource-group $AZ_RESOURCE_GROUP \
               --publisher-email "you@your-domain.com" \
               --publisher-name "Your Name"
```

> Note, depending on the SKU and resource group configuration, this operation may take 15+ min. While this running, consider quick read on [API Management Concepts](https://docs.microsoft.com/en-us/azure/api-management/api-management-key-concepts#-apis-and-operations)

### API

Each APIM [API](https://docs.microsoft.com/en-us/azure/api-management/api-management-key-concepts#-apis-and-operations) map to back-end service managed by Dapr. This demo will use a simple echo service hosted in Dapr which simply returns posted content. To define that mapping you will need to first update the [apim/api.yaml](./apim/api.yaml) file with the name of the APIM service created above:

```yaml
servers:
  - url: http://YOUR-APIM-SERVICE-NAME.azure-api.net
  - url: https://YOUR-APIM-SERVICE-NAME.azure-api.net
```

When finished, import that OpenApi definition fle into APIM service instance:

```shell
az apim api import --path / \
                   --api-id dapr \
                   --service-name $APIM_SERVICE_NAME \
                   --display-name "Demo Dapr Service API" \
                   --protocols http https \
                   --subscription-required false \
                   --specification-path apim/api.yaml \
                   --specification-format OpenApi
```

### Policy

APIM [Policies](https://docs.microsoft.com/en-us/azure/api-management/api-management-key-concepts#--policies) are defined in XML and sequentially executed on each request and/or response. To start with, we will create a global `inbound` policy for all APIs to authorize and rate-limit all requests on all operations. This simple global policy will define 2 authorization tokens and rate limit all requests to 120 calls per minute. 

> Note, for simplicity, this demo is using opaque strings which will be compared to request header. APIM supports many different ways of authentication (e.g. JWT tokens). Also, the rate limit quota we defined here is being shared across all the self-hosted gateway replicas. So, in a default configuration where there are 2 replicas, this policy would actually be 60 calls per minute.

```xml
<policies>
     <inbound>
          <check-header name="Authorization" failed-check-httpcode="401" failed-check-error-message="Not authorized" ignore-case="false">
               <value>demo1-232a021a-ac5c-4ce5-8f3e-c72559ea22d0</value>
               <value>demo2-eb5141fe-15bf-4fec-9164-cfd3ae2a80e3</value>
          </check-header>
          <set-header name="Authorization" exists-action="delete" />
          <rate-limit-by-key  
                    calls="120"
                    renewal-period="60"
                    increment-condition="@(context.Response.StatusCode == 200)"
                    counter-key="@(context.Request.IpAddress)" />
     </inbound>
     ...
</policies>
```

To apply this policy, we will first need export an Azure management API token: 

```shell
export AZ_API_TOKEN=$(az account get-access-token --resource=https://management.azure.com --query accessToken --output tsv)
```

And then apply the policy on API level API (all operations):

```shell
curl -X PUT \
     -d @apim/policy-all.json \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/policies/policy?api-version=2019-12-01"
```

To define individual operation-level policies, we will start with policy to expose the `echo` method on `echo-service` service:

```xml
<policies>
     <inbound>
          <base />
          <set-backend-service backend-id="dapr" dapr-app-id="echo-service" dapr-method="echo" />
     </inbound>
     ...
</policies>
```

To apply this policy we need to execute the similar operation like we did before on the API level but this time on only for that one operation (e.g. `/apis/dapr/operations/echo/policies/policy`): 

```shell
curl -X PUT \
     -d @apim/policy-echo.json \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/apis/dapr/operations/echo/policies/policy?api-version=2019-12-01"
```

To expose a Dapr topic we create policy that defines the `messages` topic in the `demo-events` component:

```xml
<policies>
     <inbound>
        <base />
        <publish-to-dapr 
          topic="@("demo-events/messages")" template="liquid" 
          response-variable-name="pubsub-response">{{body}}</publish-to-dapr>
        <return-response response-variable-name="pubsub-response" />
    </inbound>
     ...
</policies>
```

And once more apply that policy:


```shell
curl -X PUT \
     -d @apim/policy-pubsub.json \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/apis/dapr/operations/events/policies/policy?api-version=2019-12-01"
```

If everything goes well, all the API policy applications will returned the generated policy.

### Gateway

To create a self-hosted gateway which will be then deployed to the Kubernetes cluster, first, we need to create the `demo-apim-gateway` object in APIM:

```shell
curl -v -X PUT -d '{"properties": {"description": "Dapr Gateway","locationData": {"name": "Virtual"}}}' \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/gateways/demo-apim-gateway?api-version=2019-12-01"
```

And then map the gateway to the previously created API:

```shell
curl -v -X PUT -d '{ "properties": { "provisioningState": "created" } }' \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/gateways/demo-apim-gateway/apis/dapr?api-version=2019-12-01"
```

If everything goes well, the API returns JSON of the created objects.

## Kubernetes Configuration 

Moving now to your Kubernetes cluster...

### Dependency 

To showcase the ability to expose Dapr pub/sub API in APIM we are going to need one fo the pub/sub components configured on the cluster. One of the simplest ones is Redis. Start with adding the Redis repo to your Helm charts:

```shell
# Updating Help repos...
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

Now install Redis and wait for the deployment to complete:

> Note, for simplicity, we are deploying everything into the `default` namespace

```shell
helm install redis bitnami/redis 
kubectl rollout status statefulset.apps/redis-master
kubectl rollout status statefulset.apps/redis-slave
```

When done, deploy the Dapr component:

```shell
kubectl apply -f k8s/pubsub.yaml
```

If you are making changes to the component you will have to restart the `event-subscriber` and the `demo-apim-gateway` deployments

```shell
kubectl rollout restart deployment/event-subscriber
kubectl rollout restart deployment/demo-apim-gateway
kubectl rollout status deployment/event-subscriber
kubectl rollout status deployment/demo-apim-gateway
```


### Dapr Service 

To deploy your application as a Dapr service you just need to decorating your Kubernetes deployment template with few Dapr annotations.

```yaml
annotations:
     dapr.io/enabled: "true"
     dapr.io/app-id: "name-of-your-deployment"
```

> To learn more about Kubernetes sidecar configuration see [Dapr docs](https://github.com/dapr/docs/blob/master/concepts/configuration/README.md#kubernetes-sidecar-configuration).

For this demo we will use a pre-build Docker images of two applications:

* [http-echo-service](https://github.com/mchmarny/dapr-demos/tree/master/http-echo-service)
* [http-event-subscriber](https://github.com/mchmarny/dapr-demos/tree/master/http-event-subscriber)

The Kubernetes deployment files for both of these are:

* [k8s/echo-service.yaml](k8s/echo-service.yaml)
* [k8s/event-subscriber.yaml](k8s/event-subscriber.yaml)

Deploy both of these and check that it is ready:

```shell
kubectl apply -f k8s/echo-service.yaml
kubectl apply -f k8s/event-subscriber.yaml
kubectl get pods -l demo=dapr-apim -w
```

> Service is ready when its status is `Running` and the ready column is `2/2` (Dapr and our echo service both started)

```shell
NAME                                READY   STATUS    RESTARTS   AGE
echo-service-668986b998-v2ssp       2/2     Running   0          10m
event-subscriber-7d68b67d9d-5v7bf   2/2     Running   0          10m
```

To make sure that the event subscriber connects to the Redis service you can query the service logs

```shell
kubectl logs -l app=event-subscriber -c daprd | grep demo-events
```

You should see entries containing: 

```shell
app responded with subscriptions [{demo-events messages /messages map[]}]
app is subscribed to the following topics: [messages] through pubsub=demo-events
subscribing to topic=messages on pubsub=demo-events
```

### APIM Gateway 

To connect the self-hosted gateway to APIM service, we will need to create first a Kubernetes secret with the APIM gateway key. First, get the key from APIM API:

> Note, the maximum validity for access tokens is 30. Update the below `expiry` parameter to be withing 30 days from today

```shell
curl -X POST -d '{ "keyType": "primary", "expiry": "2020-10-10T00:00:01Z" }' \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/gateways/demo-apim-gateway/generateToken?api-version=2019-12-01"
```

Copy the content of `value` from the response and create a secret:

> Make sure the secret includes the `GatewayKey` + a space ` ` + the value of your token (e.g. `GatewayKey a1b2c3...`)

```shell
kubectl create secret generic demo-apim-gateway-token --type Opaque --from-literal value="GatewayKey paste-the-key-here"
```

Now, create a config map containing the APIM service endpoint that will be used to configure your self-hosted gateway:

```shell
kubectl create configmap demo-apim-gateway-env --from-literal \
     "config.service.endpoint=https://dapr-apim-demo.management.azure-api.net/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}?api-version=2019-12-01"
```

And finally, deploy the gateway and check that it's ready:

```shell
kubectl apply -f k8s/gateway.yaml
kubectl get pods -l app=demo-apim-gateway -w
```

> Note, the self-hosted gateway is deployed with 2 replicas to ensure availability during upgrades. 

Make sure both instances have status `Running` and container is ready `2/2` (gateway container + Dapr side-car).

```shell
NAME                                 READY   STATUS    RESTARTS   AGE
demo-apim-gateway-6dfb968f5c-cb4t7   2/2     Running   0          26s
demo-apim-gateway-6dfb968f5c-gxrrq   2/2     Running   0          26s
```

To check on the gateway logs:

```shell
kubectl logs -l app=demo-apim-gateway -c demo-apim-gateway
```

## Test

We are ready to test. Start by capturing the cluster load balancer ingress IP:

```shell
export GATEWAY_IP=$(kubectl get svc demo-apim-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

### Service Invocation 

And now, try posting a message to the APIM self-hosted gateway which will be forwarded to the backing Dapr service:

```shell
curl -X POST -d '{ "message": "hello" }' \
     -H "Content-Type: application/json" \
     -H "Authorization: demo1-232a021a-ac5c-4ce5-8f3e-c72559ea22d0" \
     "http://${GATEWAY_IP}/dapr-echo"
```

If everything is configured correctly, you should see the response from your backing Dapr service: 

```json 
{ "message": "hello" }
```

In addition, you can also check the `echo-service` logs:

```shell
kubectl logs -l app=echo-service -c service
```

### Message Publishing 

Try also posting a message to the Dapr pub/sub topic using the API exposed by APIM self-hosted gateway:

```shell
curl -X POST -d '{ "message": "hello" }' \
     -H "Content-Type: application/json" \
     -H "Authorization: demo2-eb5141fe-15bf-4fec-9164-cfd3ae2a80e3" \
     -H "Ocp-Apim-Subscription-Key: ${AZ_APIM_KEY}" \
     -H "Ocp-Apim-Trace: true" \
     "http://${GATEWAY_IP}/dapr-topic"
```

If everything is configured correctly, you should see the response from your backing Dapr service: 

```json 
{ "message": "hello" }
```

In addition, you can also check the `event-subscriber` logs:

```shell
kubectl logs -l app=event-subscriber -c service
```

### Debugging 

APIM has a very helpful way of assisting during policy development and debugging. By appending `Ocp-Apim` headers with subscription key and trace users are able to see the entire execution graph step by step. 

```shell
curl -v -X POST -d '{ "message": "hello" }' \
     -H "Content-Type: application/json" \
     -H "Authorization: demo2-eb5141fe-15bf-4fec-9164-cfd3ae2a80e3" \
     -H "Ocp-Apim-Subscription-Key: ${AZ_APIM_KEY}" \
     -H "Ocp-Apim-Trace: true" \
     "http://${GATEWAY_IP}/dapr-topic"
```

These header values, when provided during request, will result in the response including an additional `Ocp-Apim-Trace-Location` parameter with the trace URL you can use in browser to understand the APIM execution graph. An abbreviated example: 

```json
{
    "traceId": "05131509-7442-4cce-89d4-35271921ac42",
    "traceEntries": {
        "inbound": [
            {
                "source": "api-inspector",
                "timestamp": "2020-09-11T11:15:52.8238407Z",
                "elapsed": "00:00:00.0216474",
                "data": {
                    "request": {
                        "method": "POST",
                        "url": "http://x.x.x.x/dapr-topic",
                        "headers": [...]
                    }
                }
            },
            {
                "source": "api-inspector",
                "timestamp": "2020-09-11T11:15:52.8258107Z",
                "elapsed": "00:00:00.0234365",
                "data": {
                    "configuration": {
                        "api": {
                            "from": "/",
                            "to": {
                                "scheme": "https",
                                "host": "dapr-apim-demo.azure-api.net",
                                "port": 443,
                                "path": "/",
                                "queryString": "",
                                "query": {},
                                "isDefaultPort": true
                            },
                            "version": null,
                            "revision": "1"
                        },
                        "operation": {
                            "method": "POST",
                            "uriTemplate": "/dapr-topic"
                        },
                        "user": "-",
                        "product": "-"
                    }
                }
            },
            ...
            {
                "source": "check-header",
                "timestamp": "2020-09-11T11:15:52.8887185Z",
                "elapsed": "00:00:00.0863446",
                "data": {
                    "message": "Header validated.",
                    "header": {
                        "name": "Authorization",
                        "value": "xxx"
                    }
                }
            },
            ...
            {
                "source": "rate-limit-by-key",
                "timestamp": "2020-09-11T11:15:52.9002437Z",
                "elapsed": "00:00:00.0978694",
                "data": {
                    "message": "Expression was successfully evaluated.",
                    "expression": "context.Request.IpAddress",
                    "value": "x.x.x.x"
                }
            },
            ...
            {
                "source": "request-forwarder",
                "timestamp": "2020-09-11T11:15:52.9405903Z",
                "elapsed": "00:00:00.1382166",
                "data": {
                    "message": "Request is being forwarded to the backend service. Timeout set to 300 seconds",
                    "request": {
                        "method": "POST",
                        "url": "http://localhost:3500/v1.0/publish/demo-events/messages"
                    }
                }
            },
            {
                "source": "publish-to-dapr",
                "timestamp": "2020-09-11T11:15:53.1899121Z",
                "elapsed": "00:00:00.3875400",
                "data": {
                    "response": {
                        "status": {
                            "code": 200,
                            "reason": "OK"
                        },
                        "headers": [
                            {
                                "name": "Server",
                                "value": "fasthttp"
                            },
                            {
                                "name": "Date",
                                "value": "Fri, 11 Sep 2020 11:15:52 GMT"
                            },
                            {
                                "name": "Content-Length",
                                "value": "0"
                            },
                            {
                                "name": "Traceparent",
                                "value": "00-5b1f0bdfc2191742a4635a906359a7aa-196f5df2e977b00a-01"
                            }
                        ]
                    }
                }
            },
            ...
        ],
        "outbound": [...]
    }
}
```


## Cleanup 

```shell
kubectl delete -f gateway.yaml
kubectl delete -f service.yaml
kubectl delete -f k8s/pubsub.yaml
kubectl delete secret demo-apim-gateway-token
kubectl delete configmap demo-apim-gateway-env
az apim delete --name daprapimdemo --no-wait --yes
```

## Summary 

This demo illustrates how to setup the APIM service and deploy your self-hosted gateway. Using this gateway can mange access to any number of your Dapr services hosted on Kubernetes. There is a lot more that APIM can do (e.g. Discovery, Access Control, Throttling, Caching, Logging, Traces etc.). You can find out more about APIM [here](https://azure.microsoft.com/en-us/services/api-management/)