# Cheatsheet to run my new stack for Nobsmed.com

## Stack

-   `uv` workspace - in one repo I can play around with different service versions to quickly fix or compare deployment branches
-   Docker multi-stage builds (3x smaller images)
-   Backend: FastAPI (shift to [LangGraph Platform](https://www.langchain.com/langgraph-platform))
    -   async LLM calls
    -   sentence transformer to fine tune embedding models
    -   sklearn to create simple ML models to predict doc line relevance and user question topic
-   Frontend: Next.js, React components, Tailwind CSS, TypeScript
-   Search DB - sparse vector embedding search (coming soon)
-   Azure registry and container app deployment -- I got funding from Microsoft
-   Redis cache (coming soon)

---

## Principles for a 1 person team

-   Transparency over automation
    -   no github actions until CI/CD is needed due to more team members going faster
-   Simplicity - no Terraform - stick w/ Azure CLI until the infrastructure is more complex

## Steps to run my stack

1. Build the images locally
2. Push the images to cloud registry
3. Deploy cloud container based on new image in registry

Source: [Azure Container Registry for launch in Azure Container Apps.](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-tutorial-prepare-acr#create-azure-container-registry)

Assumptions:

-   Azure: Azure CLI installed and your logged into it
-   you have created a resource group in Azure (e.g., for me its `nobsmed`)

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
