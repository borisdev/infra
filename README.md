# Cheatsheet to run my Nobsmed.com stack

## My stack

-   `uv` package management for Python will replace the current Poetry package management
-   Docker multi-stage builds (3x smaller images) so no big deal pushing from laptop to cloud registry
-   Backend: FastAPI - async LLM calls, sentence transformer, sklearn
-   Frontend: Next.js, React components, Tailwind CSS, TypeScript
-   Search DB - sparse vector embedding search (coming soon)
-   Azure registry and container app deployment -- I got funding from Microsoft
-   Redis cache (coming soon)

---

## Requirements

### `uv`: Mix-and-Match Deployments for A/B Experiments

We need to quickly evaluate innovations by showing beta testers 2 cloud deployments side-by-side.

We will need `uv` python package management to run multiple deployments of the same stack, each with a different component version in the same repo.

This means each deployment has one component that varies due to the innovation, and all the other components are the same.

#### Illustration with Hypothetical Questions

**Are the new React FE Components better?**

-   deployment_v1A &#x2B05; `frontend_v1A` `backend_v1`
-   deployment_v1B &#x2B05; `frontend_v1B` `backend_v1`

**Are the new LLM prompts better?**

-   deployment v2A &#x2B05; `frontend_v2` `backend_v2A`
-   deployment v2B &#x2B05; `frontend_v2` `backend_v2B`

### Transparency over automation

-   no github actions until CI/CD is needed due to more team members going faster

### Simplicity

-   stick w/ Azure CLI until the infrastructure is more complex

## Steps to run my stack

1. Build the images locally
2. Push the images to cloud registry
3. Deploy cloud container based on new image in registry

Source: [Azure Container Registry for launch in Azure Container Apps.](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-tutorial-prepare-acr#create-azure-container-registry)

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

I prefix names with `nobs` because my company is called Nobsmed.

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
