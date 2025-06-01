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

### `uv`: Mix-and-Match Deployments for A/B Experiments

We need to quickly evaluate innovations by showing beta testers 2 cloud deployments side-by-side.

We will need `uv` python package management to run multiple deployments of the same stack, each with a different component version in the same repo.

This means each deployment has one component that varies due to the innovation, and all the other components are the same.

#### Illustration with Hypothetical Questions

**Are the new React FE Components better?**

-   \*\* deployment version.1A - `frontend_v1A` `backend_v1`
-   \*\* deployment version.1B - `frontend_v1B` `backend_v1`

**Are the new LLM prompts better?**

-   \*\* deployment version.1A - `frontend_v1` `backend_v1A`
-   \*\* deployment version.1B - `frontend_v1` `backend_v1B`

### Transparency over automation

-   no github actions until CI/CD is needed, ie., more team members going faster

### Simplicity

-   stick w/ Azure CLI until the infrastructure is more complex and requires automation

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
