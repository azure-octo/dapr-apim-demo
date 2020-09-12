# Dapr & Azure API Management Integration Demo

Dapr integration with [Azure API Management](https://azure.microsoft.com/en-us/services/api-management/) (APIM) using self-hosted gateway on Kubernetes. 

In this demo we will walk through the configuration of API Management service and its self-hosted gateway on Kubernetes. To illustrate the Dapr integration we will also review three Dapr use-cases:

* Invocation of a specific Dapr service method
* Publishing content to a Pub/Sub topic 
* Binding invocation with request content transformation

In addition, we will overview the use of APIM tracing to debug your configuration. 

> While you can accomplish everything we show here in Azure portal, to make this demo easier to reliably reproduce, we will be using only the Azure CLI and APIs.

## Prerequisite 

* [Azure account](https://azure.microsoft.com/en-us/free/)
* [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
* [Kubernetes cluster with Dapr](https://github.com/dapr/docs/blob/v0.9.0/getting-started/environment-setup.md#installing-dapr-on-a-kubernetes-cluster)
* [Helm](https://helm.sh/docs/intro/install/)

## Setup 

To make this demo easier to reproduce, start by exporting the name for your new Azure API Management (APIM) service.

> Note, the name of your API Management service instance has to be globally unique!

```shell
export APIM_SERVICE_NAME="dapr-apim-demo"
```

In addition also export the Azure [Subscription ID](https://docs.bitnami.com/azure/faq/administration/find-subscription-id/) and [Resource Group](https://docs.bitnami.com/azure/faq/administration/find-deployment-resourcegroup-id/) where you want to create that service.

```shell
export AZ_SUBSCRIPTION_ID="your-subscription-id"
export AZ_RESOURCE_GROUP="your-resource-group"
```

## Azure API Management 

We will start by configuring the Azure API Management service.

### Service Creation

Create service instance:

> The `publisher-email` and `publisher-name` are only required to receive system notifications e-mails.

```shell
az apim create --name $APIM_SERVICE_NAME \
               --subscription $AZ_SUBSCRIPTION_ID \
               --resource-group $AZ_RESOURCE_GROUP \
               --publisher-email "you@your-domain.com" \
               --publisher-name "Your Name"
```

> Note, depending on the SKU and resource group configuration, this operation may take 15+ min. While this running, consider quick read on [API Management Concepts](https://docs.microsoft.com/en-us/azure/api-management/api-management-key-concepts#-apis-and-operations)

### API Configuration

Each [API operation](https://docs.microsoft.com/en-us/azure/api-management/api-management-key-concepts#-apis-and-operations) defined in APIM will map to one Dapr API. To define these mappings you will use OpenAPI format defined in [apim/api.yaml](./apim/api.yaml) file. You will need to update the OpenAPI file with the name of the APIM service created above:

```yaml
servers:
  - url: http://<YOUR-APIM-SERVICE-NAME>.azure-api.net
  - url: https://<YOUR-APIM-SERVICE-NAME>.azure-api.net
```

When finished, import that OpenAPI definition fle into APIM service instance:

```shell
az apim api import --path / \
                   --api-id dapr \
                   --subscription $AZ_SUBSCRIPTION_ID \
                   --resource-group $AZ_RESOURCE_GROUP \
                   --service-name $APIM_SERVICE_NAME \
                   --display-name "Demo Dapr Service API" \
                   --protocols http https \
                   --subscription-required false \
                   --specification-path apim/api.yaml \
                   --specification-format OpenApi
```

### Policy Management

APIM [Policies](https://docs.microsoft.com/en-us/azure/api-management/api-management-key-concepts#--policies) are sequentially executed on each request. We will start by defining "global" policy to authorize and throttle all operation invocations on our API, then add individual policies for each operation to add specific options.

#### Global Policy

APIM policies are defined inside of inbound, outbound, and backend elements. In our case to apply policy that will authorize tokens and rate-limit all requests on all operations (before they are forwarded to Dapr API), we will place the global policy within the `inbound` section. 

> Note, while APIM supports many different ways of authentication, for simplicity, this demo will use opaque strings which will be compared to header value in each request. Also, the rate limit quota we defined here is being shared across all the replicas of self-hosted gateway. In default configuration, where there are 2 replicas, this policy would actually be half of the permitted calls per minute.

```xml
<policies>
     <inbound>
          <check-header 
               name="Authorization" 
               failed-check-httpcode="401" 
               failed-check-error-message="Not authorized" 
               ignore-case="false">
               <value>demo1-232a021a-ac5c-4ce5-8f3e-c72559ea22d0</value>
               <value>demo2-eb5141fe-15bf-4fec-9164-cfd3ae2a80e3</value>
          </check-header>
          <set-header 
               name="Authorization" 
               exists-action="delete" />
          <rate-limit-by-key  
               calls="120"
               renewal-period="60"
               increment-condition="@(context.Response.StatusCode == 200)"
               counter-key="@(context.Request.IpAddress)" />
     </inbound>
     ...
</policies>
```

To apply this [policy](apim/policy-all.json), first export the Azure management API token: 

```shell
export AZ_API_TOKEN=$(az account get-access-token --resource=https://management.azure.com --query accessToken --output tsv)
```

> If for some reason you receive token expired later just re-run this command. 

Apply that [policy](apim/policy-all.json) to all operations submit it to the Azure management API.

```shell
curl -i -X PUT \
     -d @apim/policy-all.json \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/policies/policy?api-version=2019-12-01"
```

If everything goes well, the management API will return the created policy.

#### Echo Service Policy 

The Dapr service invocation handles all the service discovery, so to invoke a specific method on any Dapr service users follow this API: 

```http
POST/GET/PUT/DELETE /v1.0/invoke/<appId>/method/<method-name>
```

To enable users to invoke the `echo` method on Dapr service with ID of `echo-service` we will create a policy that inherits the global policy (`<base />`) first, to ensure only authorize service invocation are passed to the backend Dapr API. Then to "map" the invocation we set `dapr` as the "backend-id" and define the Dapr service and method attributes to specific service ID and method name.

```xml
<policies>
     <inbound>
          <base />
          <set-backend-service 
               backend-id="dapr" 
               dapr-app-id="echo-service" 
               dapr-method="echo" />
     </inbound>
     ...
</policies>
```

To apply [this policy](apim/policy-echo.json) to the `echo` operation on our API, submit it to the Azure management API:

```shell
curl -i -X PUT \
     -d @apim/policy-echo.json \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/apis/dapr/operations/echo/policies/policy?api-version=2019-12-01"
```

If everything goes well, the management API will return the created policy.

#### Message Topic Policy 

In addition to Dapr service invocation, APIM can also be used to publish to Dapr Pub/Sub API:

```http
POST /v1.0/publish/<pubsubname>/<topic>
```

To expose the `messages` topic configured in the `demo-events` component we will start by inheriting the global policy like before, and then set the publish policy to format the request that will be passed to the Dapr Pub/Sub API:

```xml
<policies>
     <inbound>
        <base />
        <publish-to-dapr 
               topic="@("demo-events/messages")" 
               response-variable-name="pubsub-response"
        >@(context.Request.Body.As<string>())</publish-to-dapr>
        <return-response 
               response-variable-name="pubsub-response" />
    </inbound>
     ...
</policies>
```

To apply [this policy](apim/policy-message.json) to the `message` operation on our API, submit it to the Azure management API:


```shell
curl -i -X PUT \
     -d @apim/policy-message.json \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/apis/dapr/operations/message/policies/policy?api-version=2019-12-01"
```

If everything goes well, the management API will return the created policy.

#### Trigger Binding Policy 

In our final case, we are going to overview exposing the Dapr binding API.

```http
POST/PUT /v1.0/bindings/<name>
```

In contrast to the previous policies, rather than just forwarding the original request content, we are going to create a brand new request based on the content of the original request. This capability comes handy when your API needs to stay the same while the backing service evolves API evolves over time. Consider the payload expected by Dapr binding API: 

```json
{
  "data": "",
  "metadata": {
    "": "",
    "": ""
  },
  "operation": ""
}
```

To accommodate that format, out policy will also use templating engine called [liquid](https://docs.microsoft.com/en-us/azure/api-management/api-management-transformation-policies#using-liquid-templates-with-set-body).

* The `operation` will be set by the `operation` attribute in APIM `invoke-dapr-binding` policy. 
* For the `metadata` expected by DAPR, we will create a `source` element with a static `APIM` value. And for the `client-id` element the policy will select the original client request header attribute `Client-Id`.
* Finally, for `data`, we simply use the original content of the client request.

```xml
<policies>
    <inbound>
          <base />
          <invoke-dapr-binding 
               name="demo-binding" 
               operation="create" 
               template="liquid"
               response-variable-name="binding-response">
               <metadata>
                    <item key="source">APIM</item>
                    <item key="client-id">{{context.Request.Headers.Client-Id}}</item>
               </metadata>
               <data>
                    {{context.Request.Body}}
               </data>
          </invoke-dapr-binding>
          <return-response 
               response-variable-name="binding-response" />
    </inbound>
     ...
</policies>
```

To apply [this policy](apim/policy-query.json) to the `query` operation on our API, submit it to the Azure management API:

```shell
curl -i -X PUT \
     -d @apim/policy-query.json \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/apis/dapr/operations/query/policies/policy?api-version=2019-12-01"
```

> Note, the support in APIM for bindings is still rolling out across Azure regions. You can safely skip this section and just demo service invocation and topic publishing if you receive an error that `invoke-dapr-binding` is not recognize.

If everything goes well, the management API will return the created policy.

### Gateway Configuration

To create a self-hosted gateway which will be then deployed to the Kubernetes cluster, first, we need to create the `demo-apim-gateway` object in APIM:

```shell
curl -i -X PUT -d '{"properties": {"description": "Dapr Gateway","locationData": {"name": "Virtual"}}}' \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/gateways/demo-apim-gateway?api-version=2019-12-01"
```

And then map that gateway to the previously created API:

```shell
curl -i -X PUT -d '{ "properties": { "provisioningState": "created" } }' \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/gateways/demo-apim-gateway/apis/dapr?api-version=2019-12-01"
```

If everything goes well, the API returns JSON of the created objects.

## Kubernetes 

Switching now to the Kubernetes cluster...

### Dependency 

To showcase the ability to expose Dapr pub/sub and binding APIs in APIM, we are going to need Dapr components configured on the cluster. 

> Note, to keep things simple for this demo we will use Redis as both pub/sub and binding backing service but you can substitute it for any one of the 75+ different components Dapr offers today.

Start with adding the Redis repo to your Helm charts:

```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

And install Redis and wait for the deployment to complete:

> Note, for simplicity, we are deploying everything into the `default` namespace.

```shell
helm install redis bitnami/redis 
kubectl rollout status statefulset.apps/redis-master
kubectl rollout status statefulset.apps/redis-slave
```

### Components 

When done, deploy the Dapr components:

```shell
kubectl apply -f k8s/pubsub.yaml
kubectl apply -f k8s/binding.yaml
```

> Note, if you updated components after deploying the gateway you will need to restart the deployments.

```shell
kubectl rollout restart deployment/event-subscriber
kubectl rollout status deployment/event-subscriber
kubectl rollout restart deployment/demo-apim-gateway
kubectl rollout status deployment/demo-apim-gateway
```

To check if the components were registered correctly in Dapr, inspect the `daprd` logs in `demo-apim-gateway` pod for `demo-events` and `demo-binding`:

```shell
kubectl logs -l app=demo-apim-gateway -c daprd --tail=200
```

### Dapr Services 

To deploy your application as a Dapr service you will need to decorate your Kubernetes deployment template with few Dapr annotations.

```yaml
annotations:
     dapr.io/enabled: "true"
     dapr.io/app-id: "name-of-your-deployment"
```

> To learn more about Kubernetes sidecar configuration see [Dapr docs](https://github.com/dapr/docs/blob/master/concepts/configuration/README.md#kubernetes-sidecar-configuration).

For this demo we will use a pre-build Docker images of two applications: [gRPC Echo Service](https://github.com/mchmarny/dapr-demos/tree/master/grpc-echo-service) and [HTTP Event Subscriber Service](https://github.com/mchmarny/dapr-demos/tree/master/http-event-subscriber). The Kubernetes deployments for both of these are located here:

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

### Self-hosted APIM Gateway 

To connect the self-hosted gateway to APIM service, we will need to create a Kubernetes secret with the APIM gateway key. Start by getting the key from APIM API:

> Note, the maximum validity for access tokens is 30. Update the below `expiry` parameter to be withing 30 days from today

```shell
curl -i -X POST -d '{ "keyType": "primary", "expiry": "2020-10-10T00:00:01Z" }' \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/gateways/demo-apim-gateway/generateToken?api-version=2019-12-01"
```

Copy the content of `value` from the response and create a secret:

```shell
kubectl create secret generic demo-apim-gateway-token --type Opaque --from-literal value="GatewayKey paste-the-key-here"
```

> Make sure the secret includes the `GatewayKey` and a space ` ` as well as the value of your token (e.g. `GatewayKey a1b2c3...`)

Now, create a config map containing the APIM service endpoint that will be used to configure your self-hosted gateway:

```shell
kubectl create configmap demo-apim-gateway-env --from-literal \
     "config.service.endpoint=https://dapr-apim-demo.management.azure-api.net/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}?api-version=2019-12-01"
```

And finally, deploy the gateway and check that it's ready:

```shell
kubectl apply -f k8s/gateway.yaml
kubectl get pods -l app=demo-apim-gateway
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

## Usage (API Test)

With APIM configured and self-hosted gateway deployed we are ready to test. Start by capturing the cluster load balancer ingress IP:

```shell
export GATEWAY_IP=$(kubectl get svc demo-apim-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

### Service Invocation 

To invoke the backing gRPC service over Dapr API exposed by APIM run:

```shell
curl -i -X POST -d '{ "message": "hello" }' \
     -H "Content-Type: application/json" \
     -H "Authorization: demo1-232a021a-ac5c-4ce5-8f3e-c72559ea22d0" \
     "http://${GATEWAY_IP}/echo"
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

To post a message to the Dapr Pub/Sub API exposed on APIM run:

```shell
curl -i -X POST \
     -d '{ "content": "hello" }' \
     -H "Content-Type: application/json" \
     -H "Authorization: demo2-eb5141fe-15bf-4fec-9164-cfd3ae2a80e3" \
     "http://${GATEWAY_IP}/message"
```

If everything is configured correctly, you will see `200` status code in the header, indicating the message was successfully delivered to the Dapr API.

You can also check the `event-subscriber` logs:

```shell
kubectl logs -l app=event-subscriber -c service
```

There should be an entry similar to this: 

```shell
event - PubsubName:demo-events, Topic:messages, ID:24f0e6f0-ab29-4cd6-8617-6c6c36ac1171, Data: map[message:hello]
```

### Binding Triggering

To trigger Dapr binding API exposed by APIM run:

```shell
curl -X POST -d '{ "query": "serverless", "lang": "en", "result": "recent" }' \
     -H "Content-Type: application/json" \
     -H "Authorization: demo3-fg8754fe-75eb-7cef-9876-dcf3ae2a99f1" \
     -H "Client-Id: id-123456789" \
     "http://${GATEWAY_IP}/query"
```

If everything is configured correctly, you will see `200` status code in the header indicating the binding was successfully triggered on the Dapr API.

## Summary 

This demo illustrates how to setup the APIM service and deploy your self-hosted gateway. Using this gateway can mange access to any number of your Dapr services hosted on Kubernetes. You can find out more about all the features of APIM (e.g. Discovery, Caching, Logging etc.) [here](https://azure.microsoft.com/en-us/services/api-management/).

### Debugging 

APIM provides tracing which is helpful to debug policies. To take advantage of this feature you will first need a subscription key: 

```shell
curl -i -H POST -d '{}' -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/subscriptions/master/listSecrets?api-version=2019-12-01"
```

The response will include both the primary and secondary keys. Copy one of them and paste it into the `Ocp-Apim-Subscription-Key` header parameter. That along with `Ocp-Apim-Trace: true` in your request header will tell APIM to capture trace for your invocation:

```shell
curl -v -X POST -d '{ "message": "hello" }' \
     -H "Content-Type: application/json" \
     -H "Authorization: demo2-eb5141fe-15bf-4fec-9164-cfd3ae2a80e3" \
     -H "Ocp-Apim-Subscription-Key: YOUR-SUBSCRIPTION-KEY-HERE" \
     -H "Ocp-Apim-Trace: true" \
     "http://${GATEWAY_IP}/message"
```

The response of our invocation will now also include the `Ocp-Apim-Trace-Location` header parameter which will hold the URL to where you can view your trace. Paste that URL into browser to get the full trace: 

```json
{
    "traceId": "05131509-7442-4cce-89d4-35271921ac42",
    "traceEntries": {
        "inbound": [...],
        "outbound": [...]
    }
}
```

The details of the trace are long. For example you will be able to see the message that was forwarded to Dapr API and what was its response:

```json 
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
}
...
```


## Cleanup 

```shell
kubectl delete -f k8s/gateway.yaml

kubectl delete -f k8s/echo-service.yaml
kubectl delete -f k8s/event-subscriber.yaml

kubectl delete -f k8s/pubsub.yaml
kubectl delete -f k8s/binding.yaml

kubectl delete secret demo-apim-gateway-token
kubectl delete configmap demo-apim-gateway-env

az apim delete --name $APIM_SERVICE_NAME --no-wait --yes
```

