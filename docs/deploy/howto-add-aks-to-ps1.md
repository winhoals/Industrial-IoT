# Adding an AKS cluster with Azure Industrial IoT components on top of script deployment <!-- omit in toc -->

[Home](readme.md)

This article explains how to add an AKS cluster and deploy components of Azure Industrial IoT Platform using
a Helm chart on top of a platform deployed through either `deploy.cmd` or `deploy.sh` scripts.

## Table of contents <!-- omit in toc -->

* [Introduction](#introduction)
* [Prerequisites](#prerequisites)
* [Deployment Steps](#deployment-steps)
  * [Create an AKS cluster](#create-an-aks-cluster)
  * [Create a Public IP address](#create-a-public-ip-address)
  * [Get credentials for kubectl](#get-credentials-for-kubectl)
  * [Deploy ingress-nginx Helm chart](#deploy-ingress-nginx-helm-chart)
  * [Deploy cert-manager Helm chart](#deploy-cert-manager-helm-chart)
  * [Create ClusterIssuer resource](#create-clusterissuer-resource)
  * [Stop App Service resources](#stop-app-service-resources)
  * [Deploy azure-industrial-iot Helm chart](#deploy-azure-industrial-iot-helm-chart)
    * [Using configuration in Azure Key Vault](#using-configuration-in-azure-key-vault)
    * [Passing Azure resource details through YAML file](#passing-azure-resource-details-through-yaml-file)
    * [Installing `azure-industrial-iot` Helm chart](#installing-azure-industrial-iot-helm-chart)
      * [Installing the chart from GitHub repository](#installing-the-chart-from-github-repository)
      * [Installing the chart from Helm chart repository](#installing-the-chart-from-helm-chart-repository)
  * [Check status of deployed resources](#check-status-of-deployed-resources)
  * [Enable Prometheus metrics scraping](#enable-prometheus-metrics-scraping)
  * [Update Redirect URIs of web App Registration](#update-redirect-uris-of-web-app-registration)
  * [Remove App Service and App Service Plan resources](#remove-app-service-and-app-service-plan-resources)
  * [Access Engineering Tool and Swagger UIs](#access-engineering-tool-and-swagger-uis)

## Introduction

`deploy.cmd` or `deploy.sh` scripts
are deploying components of Azure Industrial IoT Platform into two instances of Azure App Services, and as
such they do not provide high degree of scalability and high-availability. We recommend using those scripts
for PoC and demo purposes. For production scenarios, we recommend running component of Azure Industrial IoT
platform in Kubernetes cluster. This article will guide you through the steps of making your existing
deployment into a more production-ready one.

Please note that we also provide `Microsoft.Azure.IIoT.Deployment (Preview)` command-line application, which
similar to deployment scripts creates an instance of Azure Industrial IoT Platform. In contrast to
deployment scripts, it deploys components of Azure Industrial IoT Platform into an AKS cluster. Please use
that application if you are starting from scratch:

* Deploying Azure Industrial IoT Platform to [Azure Kubernetes Service (AKS)](howto-deploy-aks.md) as
  production solution.

## Prerequisites

* We are going to assume that you have already successfully run `deploy.cmd` or `deploy.sh` scripts and
  have the `.env` file generated by them. We will also need you to note the names of three App Registrations
  that have been created by those scripts. They will have the same name prefix, which would be name of
  application that you set. And they will have the following suffixes:

  * `-client`, we will refer to this one as `client` App Registration
  * `-web`, we will refer to this one as `web` App Registration
  * `-service`, we will refer to this one as `service` App Registration

  If you first need to run deployment scripts, please use this documentation:

  [Azure Industrial IoT Platform and Simulation demonstrator using the deployment script](howto-deploy-all-in-one.md)

* Azure CLI: [Install the Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli)
* `kubectl`: You can run the following command to install `kubectl` using Azure CLI:

  ```bash
  az aks install-cli
  ```

* Helm 3: Please follow the steps in the official documentation to install Helm CLI:
  [Installing Helm](https://helm.sh/docs/intro/install/)

  Alternatively, you can run the following command to install `helm` using Azure CLI:

  ```bash
  az acr helm install-cli --client-version "3.7.1"
  ```

## Deployment Steps

The steps to create AKS cluster with static public IP address bellow are based on the following tutorial:

* [Create an ingress controller with a static public IP address in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/azure/aks/ingress-static-ip)

### Create an AKS cluster

Go to the resource group that houses your deployment of Azure Industrial IoT Platform and create and AKS
cluster in it. Please follow the steps in this tutorial and note the steps below it:

* [Create an AKS cluster using Azure portal](https://docs.microsoft.com/azure/aks/kubernetes-walkthrough-portal#create-an-aks-cluster)

1. On the **Basics** page, give a name to your cluster, let's say `aks-cluster`.
2. On the **Node pools** page, keep the default to go further.
3. on the **Authentication** page, select `System-assigned managed identity` as `Authentication method`.
4. On the **Networking** page, you can either use the default or set `Network configuration` to `Advanced`
  and add the cluster to and existing VNet.
5. On the **Integrations** page, make sure that `Container monitoring` is set to `Enabled` and set
  `Log Analytics workspace` to an instance in your resource group.
6. On the **Tags** page, you can add custom tags.
7. On the **Review + create** page, review cluster details and create it.

Now please wait for the AKS cluster to be created. That might take up to 15 minutes.

### Create a Public IP address

After AKS cluster has been created, we will create a Public IP address resource in the resource group managed
by the AKS cluster. Please check for it the list of your resource groups, it will have the following naming
format:

`MC_<myResouceGroup>_<aksClusterName>_<regionName>`

Go to it and create a Public IP address resource using Azure portal and apply the following changes:

On the **Create public IP address** page:

1. Set `SKU` to `Standard`.
2. Set a `Name` for the resource, let's say `aks-cluster-ip`.
3. Set a `DNS name label`, let's say `aks-cluster-ip`. This will determine URL of our services.
4. Make sure it uses the same `Location` as your AKS cluster and the rest of your resources.

Now please wait for the Public IP address to be created.

Once it is created, go to the **Overview** page of the Public IP address and copy the following details,
we will need them when deploying NGINX Ingress Controller and Azure Industrial IoT Helm charts:

* `IP address`
* `DNS name`

### Get credentials for kubectl

Provide your resource group and AKS cluster names:

```bash
az aks get-credentials --resource-group myResourceGroup --name myAksCluster
```

### Deploy ingress-nginx Helm chart

Create `ingress-nginx` namespace:

```bash
kubectl create namespace ingress-nginx
```

Add `ingress-nginx` Helm chart repository:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Create `ingress-nginx.yaml` file that we will use for deploying the Helm chart based on the template below.

Apply the following changes:

1. Use IP address of the Public IP address resource that we created earlier instead of `20.50.19.169`
2. Use DNS name label that you set when creating the Public IP address resource instead of
  `aks-cluster-ip`. Use the value that you set when creating the resource, which is only the
  **first part** of the hostname and **not** the whole hostname.

```yaml
controller:
  replicaCount: 2
  service:
    loadBalancerIP: 20.50.19.169
    annotations:
      service.beta.kubernetes.io/azure-dns-label-name: aks-cluster-ip
  config:
    compute-full-forwarded-for: true
    use-forwarded-headers: true
    proxy-buffer-size: "32k"
    client-header-buffer-size: "32k"
  metrics:
    enabled: true
defaultBackend:
  enabled: true
```

Install `ingress-nginx/ingress-nginx` Helm chart using `ingress-nginx.yaml` file created above.

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --version 4.0.19 -f ingress-nginx.yaml
```

Documentation for `ingress-nginx/ingress-nginx` Helm chart can be found [here](https://artifacthub.io/packages/helm/ingress-nginx/ingress-nginx).

### Deploy cert-manager Helm chart

Create `cert-manager` namespace and label it to disable resource validation:

```bash
kubectl create namespace cert-manager
```

Add `jetstack` Helm chart repository:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

Install `jetstack/cert-manager` Helm chart:

```bash
helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.8.0 --set installCRDs=true
```

Documentation for `jetstack/cert-manager` Helm chart can be found [here](https://artifacthub.io/packages/helm/jetstack/cert-manager).

### Create ClusterIssuer resource

Create a `ClusterIssuer` resource using the following `cluster-issuer.prod.yaml` file.
You can set your email address in `ClusterIssuer` so that letsencrypt informs you if they find issues with
the certificate that was issued to you. To do that uncomment `email:` line and set the value.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    # email: <your-email-address@something.com>
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx
```

Run the command:

```bash
kubectl apply -f cluster-issuer.prod.yaml
```

### Stop App Service resources

Before deploying `azure-industrial-iot` Helm chart, please go to your resource group, find App Service
resources and stop them. This will ensure that there are no two versions of the platform running at the
same time as that may cause some issues with module deployments in IoT Hub.

### Deploy azure-industrial-iot Helm chart

Create `aiiot` namespace:

```bash
kubectl create namespace aiiot
```

Now you will need to decide whether you want to use configuration that has been pushed to Azure Key Vault
as secrets by deployment scripts or pass details of all required Azure resources through the chart. Based
on that the YAML values files for `azure-industrial-iot` Helm chart will look a bit differently. Below we
will go over the values files for those two cases and then later use one of them to install the chart.

#### Using configuration in Azure Key Vault

To use the configuration in Azure Key Vault we need to set up Access Policies for service principal of
`service` App Registration so that microservice have permission to read secrets from Azure Key Vault.

To do that:

1. In Azure portal, go the the Key Vault resource that has been created by the deployment script.
2. On the left panel, go to **Access policies** page under **Settings**.
3. Click on **+ Add Access Policy**.
4. Add the following permissions:

    * Key Permissions: `get`, `list`, `sign`, `unwrapKey`, `wrapKey`, `create`
    * Secret Permissions: `get`, `list`, `set`, `delete`
    * Certificate Permissions: `get`, `list`, `update`, `create`, `import`

5. Click on **Select principal**, and type in the name of `service` App Registration in the search bar,
  then select it when it is found.

6. Click on **Add** button which will return you to **Access policies** page.
7. Click on **Save** on the top bar to confirm addition of the policy.

We would also need to enable configuration loading from Azure Key Vault by setting `loadConfFromKeyVault`
to `true` for the Helm chart. Then, only the following parameters of `azure.*` parameter group would be
required by the chart:

* `azure.tenantId`
* `azure.keyVault.uri`
* `azure.auth.servicesApp.appId`
* `azure.auth.servicesApp.secret`

Please check the [documentation of `azure-industrial-iot` Helm chart](../../deploy/helm/azure-industrial-iot/README.md)
for more details about the `loadConfFromKeyVault` and other parameter that are set in this case. In the end,
`aiiot.yaml` value file would look something like the one below.

Note the following values in the YAML file:

* Use hostname you got from `DNS name` of Public IP address resource for:
  * `externalServiceUrl`, should include `https://` protocol
  * `deployment.ingress.tls[0].hosts[0]` and `deployment.ingress.hostName`

* Values of `azure.auth.servicesApp.appId` and `azure.auth.servicesApp.secret` can be found in
  `pcs-auth-service-appid` and `pcs-auth-service-secret` secrets in Azure Key Vault.

* You can get values of `azure.tenantId` and `azure.keyVault.uri` either from Azure portal or corresponding
  `pcs-` secrets in Azure Key Vault.

```yaml
image:
  tag: 2.8.3

loadConfFromKeyVault: true

azure:
  tenantId: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX

  keyVault:
    uri: https://keyvault-XXXXXX.vault.azure.net/

  auth:
    servicesApp:
      appId: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
      secret: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=

externalServiceUrl: https://aks-cluster-ip.westeurope.cloudapp.azure.com

deployment:
  microServices:
    engineeringTool:
      enabled: true

  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
      nginx.ingress.kubernetes.io/affinity: cookie
      nginx.ingress.kubernetes.io/session-cookie-name: affinity
      nginx.ingress.kubernetes.io/session-cookie-expires: "14400"
      nginx.ingress.kubernetes.io/session-cookie-max-age: "14400"
      nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
      nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
      cert-manager.io/cluster-issuer: letsencrypt-prod
    tls:
    - hosts:
      - aks-cluster-ip.westeurope.cloudapp.azure.com
      secretName: tls-secret
    hostName: aks-cluster-ip.westeurope.cloudapp.azure.com
```

> **NOTE**: Please note that we have used `2.8.3` as the value of `image:tag` configuration parameter
> above. That will result in `2.8.3` version of microservices and edge modules to be deployed. If you want
> to deploy a different version of the platform, please specify it as the value of `image:tag` parameter.

#### Passing Azure resource details through YAML file

If you decide to pass all Azure resource details through YAML file, please follow
[documentation of `azure-industrial-iot` Helm chart](../../deploy/helm/azure-industrial-iot/README.md)
to get the parameters. In the end, `aiiot.yaml` value file would look something like the one below.

In this case as well, we will have to set up Access Policies for service principal of `service` App
Registration in Azure Key Vault so that Engineering Tool microservice is able to fetch the
`dataprotection` key from  Azure Key Vault. That is required for proper functionality of
[ASP.NET Core Data Protection](https://docs.microsoft.com/aspnet/core/security/data-protection/configuration/overview?view=aspnetcore-3.1#protectkeyswithazurekeyvault)
feature.

To do that:

1. In Azure portal, go the the Key Vault resource that has been created by the deployment script.
2. On the left panel, go to **Access policies** page under **Settings**.
3. Click on **+ Add Access Policy**.
4. Add the following permissions:

    * Key Permissions: `get`, `list`, `sign`, `unwrapKey`, `wrapKey`, `create`

5. Click on **Select principal**, and type in the name of `service` App Registration in the search bar,
  then select it when it is found.

6. Click on **Add** button which will return you to **Access policies** page.
7. Click on **Save** on the top bar to confirm addition of the policy.

Note the following values in the YAML file:

* Use hostname you got from `DNS name` of Public IP address resource for:
  * `externalServiceUrl`, should include `https://` protocol
  * `deployment.ingress.tls[0].hosts[0]` and `deployment.ingress.hostName`

* All `azure.*` values can be obtained either from Azure portal or from corresponding `pcs-` secrets in
  Azure Key Vault.

```yaml
image:
  tag: 2.8.3

azure:
  tenantId: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX

  iotHub:
    eventHub:
      endpoint: Endpoint=sb://iothub-ns-iothub-XXX-XXXXXXX-XXXXXXXXXX.servicebus.windows.net/;SharedAccessKeyName=iothubowner;SharedAccessKey=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=;EntityPath=iothub-XXXXXX

      consumerGroup:
        events: events
        telemetry: telemetry
        onboarding: onboarding

    sharedAccessPolicies:
      iothubowner:
        connectionString: HostName=iothub-XXXXXX.azure-devices.net;SharedAccessKeyName=iothubowner;SharedAccessKey=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=

  cosmosDB:
    connectionString: AccountEndpoint=https://documentdb-XXXXXX.documents.azure.com:443/;AccountKey=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==;

  storageAccount:
    connectionString: DefaultEndpointsProtocol=https;AccountName=storageXXXXXX;AccountKey=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==;EndpointSuffix=core.windows.net

  eventHubNamespace:
    sharedAccessPolicies:
      rootManageSharedAccessKey:
        connectionString: Endpoint=sb://eventhubnamespace-XXXXXX.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=

    eventHub:
      name: eventhub-XXXXXX

      consumerGroup:
        telemetryUx: telemetry_ux

  serviceBusNamespace:
    sharedAccessPolicies:
      rootManageSharedAccessKey:
        connectionString: Endpoint=sb://sb-XXXXXX.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=

  keyVault:
    uri: https://keyvault-XXXXXX.vault.azure.net/

  applicationInsights:
    instrumentationKey: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX

  logAnalyticsWorkspace:
    id: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
    key: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==

  signalR:
    connectionString: Endpoint=https://hubXXXXXX.service.signalr.net;AccessKey=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=;Version=1.0;
    serviceMode: Default

  auth:
    servicesApp:
      appId: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
      secret: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=
      audience: https://opcwalls.onmicrosoft.com/XXXXXXXXXXXXXX-service

    clientsApp:
      appId: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
      secret: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=

externalServiceUrl: https://aks-cluster-ip.westeurope.cloudapp.azure.com

deployment:
  microServices:
    engineeringTool:
      enabled: true

  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
      nginx.ingress.kubernetes.io/affinity: cookie
      nginx.ingress.kubernetes.io/session-cookie-name: affinity
      nginx.ingress.kubernetes.io/session-cookie-expires: "14400"
      nginx.ingress.kubernetes.io/session-cookie-max-age: "14400"
      nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
      nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
      cert-manager.io/cluster-issuer: letsencrypt-prod
    tls:
    - hosts:
      - aks-cluster-ip.westeurope.cloudapp.azure.com
      secretName: tls-secret
    hostName: aks-cluster-ip.westeurope.cloudapp.azure.com
```

> **NOTE**: Please note that we have used `2.8.3` as the value of `image:tag` configuration parameter
> above. That will result in `2.8.3` version of microservices and edge modules to be deployed. If you want
> to deploy a different version of the platform, please specify it as the value of `image:tag` parameter.

#### Installing `azure-industrial-iot` Helm chart

##### Installing the chart from GitHub repository

Here we will install `azure-industrial-iot` Helm chart from the root of the GitHub repo. This deployment
method should be used for development purposes. For production deployments please check next section on
[installing the chart from Helm chart repository](#installing-the-chart-from-helm-chart-repository).

For installing the chart, please use `aiiot.yaml` that you created above and run the following command:

```bash
helm install aiiot --namespace aiiot .\deploy\helm\azure-industrial-iot\ -f aiiot.yaml
```

`aiiot.yaml` that we created above is for `0.4.x` or `0.3.2` versions of the Helm chart. Please modify it
accordingly if you want to install a different version of the chart. Please check documentation of each
version for a list of applicable values for that specific version. Documentation links of different versions
of the chart can be found in [Deploying Azure Industrial IoT Platform Microservices Using Helm](howto-deploy-helm.md).

##### Installing the chart from Helm chart repository

You can also install `azure-industrial-iot` Helm charts from one of Helm repositories that we publish to.
For that you would first add Helm repository and then install the chart from there as shown bellow.

```bash
helm repo add azure-iiot https://azure.github.io/Industrial-IoT/helm
helm repo update
helm install aiiot azure-iiot/azure-industrial-iot --namespace aiiot --version 0.4.3 -f aiiot.yaml
```

Please note that the version of the chart in GitHub repo will be ahead of chart versions that we publish
to Helm repositories.

Details of both Helm chart repositories that we publish to can be found in [Helm Repositories](howto-deploy-helm.md#helm-repositories).

### Check status of deployed resources

To check status of deployed resources one can use Kubernetes Dashboard. Please follow the steps
[here](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) to deploy and then access
Kubernetes Dashboard. Alternatively, use `kubectl` command to get status of Kubernetes resources such as pods,
deployments or services.

### Enable Prometheus metrics scraping

To enable prometheus metrics scraping let's use `04_oms_agent_configmap.yaml` file that we have in the repo:

```bash
kubectl apply -f .\deploy\src\Microsoft.Azure.IIoT.Deployment\Resources\aks\04_oms_agent_configmap.yaml
```

### Update Redirect URIs of web App Registration

Now we will need to go to `web` App Registration and change its Redirect URIs to point to the URL of
Public IP address.

For that do the following:

1. In Azure Portal to to App Registration page and find `web` entry.
2. On the left side find **Authentication** page under **Manage** group, go there.
3. You need to add the following entries in **Redirect URIs** of **Web** platform substituting
  `aks-cluster-ip.westeurope.cloudapp.azure.com` with the hostname that you got from `DNS name` of Public
  IP address resource:

   * `https://aks-cluster-ip.westeurope.cloudapp.azure.com/frontend/signin-oidc`
   * `https://aks-cluster-ip.westeurope.cloudapp.azure.com/registry/swagger/oauth2-redirect.html`
   * `https://aks-cluster-ip.westeurope.cloudapp.azure.com/twin/swagger/oauth2-redirect.html`
   * `https://aks-cluster-ip.westeurope.cloudapp.azure.com/history/swagger/oauth2-redirect.html`
   * `https://aks-cluster-ip.westeurope.cloudapp.azure.com/edge/publisher/swagger/oauth2-redirect.html`
   * `https://aks-cluster-ip.westeurope.cloudapp.azure.com/events/swagger/oauth2-redirect.html`
   * `https://aks-cluster-ip.westeurope.cloudapp.azure.com/publisher/swagger/oauth2-redirect.html`

4. You should also delete `*.azurewebsites.net` entries from this list, as those point to containers that
  were running in App Service resources that we have stopped in previous steps. We will delete those
  instances in the next step.
5. Click **Save** to save the changes.

Now you should be able to go to Engineering Tool and Swagger UIs of microservices and authenticate there.

### Remove App Service and App Service Plan resources

Now find App Service instances that were created by the deployment script and delete them. After that find
App Service Plan resource and delete that one as well. You do not need those as now microservices are running
in the AKS cluster instead of the App Service instances

### Access Engineering Tool and Swagger UIs

Now you should be able to access Engineering Tool and Swagger UIs of microservices. To do that please
substitute `aks-cluster-ip.westeurope.cloudapp.azure.com` with the hostname that you got from `DNS name` of
Public IP address resource.

| Service                                | URL                                                                                      |
|----------------------------------------|------------------------------------------------------------------------------------------|
| Engineering Tool                       | `https://aks-cluster-ip.westeurope.cloudapp.azure.com/frontend/`                         |
| OPC Registry Swagger UI                | `https://aks-cluster-ip.westeurope.cloudapp.azure.com/registry/swagger/index.html`       |
| OPC Twin Swagger UI                    | `https://aks-cluster-ip.westeurope.cloudapp.azure.com/twin/swagger/index.html`           |
| OPC Historian Access Swagger UI        | `https://aks-cluster-ip.westeurope.cloudapp.azure.com/history/swagger/index.html`        |
| OPC Publisher Swagger UI               | `https://aks-cluster-ip.westeurope.cloudapp.azure.com/publisher/swagger/index.html`      |
| Publisher Jobs Orchestrator Swagger UI | `https://aks-cluster-ip.westeurope.cloudapp.azure.com/edge/publisher/swagger/index.html` |
| Events Swagger UI                      | `https://aks-cluster-ip.westeurope.cloudapp.azure.com/events/swagger/index.html`         |
