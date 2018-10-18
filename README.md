# AzureContainers

A package for working with Azure Container Registry (ACR), Azure Kubernetes Service (AKS) and Azure Container Instances (ACI). Extends the Azure Resource Manager interface provided by the [AzureRMR](https://github.com/Hong-Revo/AzureRMR) package.

AzureContainers lets you build and deploy containerised services in R, using Docker and Kubernetes. For full functionality, you should have [Docker](https://docs.docker.com/install/) installed, as well as the [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) commandline tool. Otherwise it is relatively lightweight, requiring neither Powershell nor Python.

Note that AzureContainers can talk to any Docker registry that uses the [V2 HTTP API](https://docs.docker.com/registry/spec/api/), not just those created via ACR. Similarly, it can interface with Kubernetes clusters anywhere, not just those created via AKS.

## Example workflow

Here is a sample R workflow to package up an R model as a container, deploy it to a Kubernetes cluster, and expose it as a service.

```r
# login to Azure ---
az <- AzureRMR::az_rm$new("<tenant_id>", "<app_id>", "<secret>")
resgroup <- az$
    get_subscription("<subscription_id>")$
    create_resource_group("myresgroup", location="australiaeast")


# create container registry ---
acr <- resgroup$create_acr("myacr", location="australiaeast")


# create Docker image from a predefined Dockerfile ---
call_docker("build -t newcontainer .")


# get registry endpoint, upload image ---
reg <- acr$get_docker_registry()
reg$push("newcontainer")


# create Kubernetes cluster with 2 nodes ---
aks <- resgroup$create_aks("myakscluster",
    location="australiaeast",
    agent_pools=aks_pools("pool1", 2, "Standard_DS2_v2", "Linux"))


# get cluster endpoint, deploy from ACR to AKS with predefined yaml definition file ---
clus <- aks$get_cluster()
clus$create_registry_secret(reg, email="email@example.com")
clus$create("model1.yaml")
clus$get("service")


# test the deployment ---
response <- httr::POST("http://<ip_address>:8000/score", body=list(df=mtcars[1:10,]), encode="json")
jsonlite::fromJSON(httr::content(response, as="text"), flatten=TRUE)
#[,1]
#[1,] 22.5776
#[2,] 22.1081
#[3,] 26.2834
#[4,] 21.1876
#[5,] 17.6451
#[6,] 20.3864
#[7,] 14.3661
#[8,] 22.4849
#[9,] 24.3762
#[10,] 18.7343


# delete the deployment ---
clus$delete("service", "model1-svc")
clus$delete("deployment", "model1")

```