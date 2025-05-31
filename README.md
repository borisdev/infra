# Cheatsheet to run my Nobsmed.com stack

I prefix names with `nobs` because my company is called Nobsmed.

## My stack

-   Poetry (later switching to uv because I want a workspace for my mix and match experiments - see below
-   Docker multi-stage builds (3x smaller images)
-   backend using FastAPI - perfect fit for my async LLM and pandas, sentence transformers, sklearn data science and ML experiments
-   frontend using Next.js --- seems to be the best....my weakness is frontend
-   OpenSearch for search DB -- thinking of switching to sparse vector search
-   Azure registry and container app deployment -- I got funding from Microsoft for this, so I am using Azure

## My stack down the road

-   Azure Search DB ?
-   Redis for caching

---

## Requirements

### Mix and match experiments

-   quickly compare two different frontend versions without changing the backend API
-   quickly compare two different backend LLM responses without changing the frontend

### Transparency over automation

-   easy to debug, ie., transparent, no magic (stay away from github actions until its needed)
-   local and cloud are the same
-   as simple as possible, I am a solo developer (stick w/ Azure CLI, no Terraform, no Pulumi, no CDK)

## Steps to run my stack

Build the images and then [push the images to Azure Container Registry for launch in Azure Container Apps.](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-tutorial-prepare-acr#create-azure-container-registry)

Assumptions:

-   you have an Azure account
-   you have the Azure CLI installed
-   you have Docker installed
-   you have Python 3.10+ installed
-   you have Poetry installed
-   you have Node.js 18+ installed
-   you have the Azure CLI logged in to your account
-   you have the Azure CLI configured to use the correct subscription (if you have multiple subscriptions)
-   you have created a resource group in Azure (e.g., `nobsmed`)

## Pre-requisites steps

```console
# create the registry

az acr create --resource-group nobsmed --name nobsregistry --sku Basic

# sanity check

az acr show --name nobsregistry --query loginServer --output table
Result
--------------------------
nobsmedregistry.azurecr.io
```

---

## Ongoing launch steps

```bash
# Build backend and frontend docker images:

docker build --tag nobs_backend --file docker/Dockerfile .
docker build --tag nobs_frontend --file docker/Dockerfile .

# Tag the images for Azure Container Registry

docker tag nobs_backend nobsregistry.azurecr.io/nobs_backend:v1

# Sanity check

docker images
REPOSITORY                             TAG       IMAGE ID       CREATED        SIZE
nobsregistry.azurecr.io/nobs_backend      v1        585a4696bedd   44 hours ago   197MB
nobs_backend                           latest    585a4696bedd   43 hours ago   197MB
nobs_frontend                          latest    585a4696bedd   43 hours ago   197MB

# Login to Azure Container Registry

az acr login --name nobsregistry

# Push image to Azure Container Registry

docker push nobsregistry.azurecr.io/nobs_backend:v1

**# Sanity check .....you see the output below**

The push refers to repository [nobsregistry.azurecr.io/nobs_backend]
5f70bf18a086: Preparing
7d577052c02c: Preparing
ac42807ce093: Preparing
d54c60f73cbb: Preparing
d559dc6e6c29: Preparing
8f697a207321: Waiting


# Sanity check that the image is in the registry

az acr repository list --name nobsregistry
[
"nobs_backend",
]
```

```console
source azure_env
source azure_deploy

az containerapp create \
  --name $API_NAME \
  --resource-group $RESOURCE_GROUP \
  --environment $ENVIRONMENT \
  --image $ACR_NAME.azurecr.io/$API_NAME \
  --target-port 8080 \
  --ingress external \
  --registry-server $ACR_NAME.azurecr.io \
  --user-assigned "$IDENTITY_ID" \
  --registry-identity "$IDENTITY_ID" \
  --query properties.configuration.ingress.fqdn

az container delete --name sampleapp --resource-group nobsmed
```
