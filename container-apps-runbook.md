# Deploy & Manage Apps on Azure Container Apps — Runbook (3 labs)

The three labs from the MS Learn path **Deploy and manage apps on Azure Container Apps** (course chapter 2 of AI-200): **deploy**, **manage**, and **scale** containers on ACA. Every command has a `#` comment explaining what it does. Instead of running the lab's `azdeploy.py` script, I run its steps by hand so each command is understood.

> Keep one terminal per lab so the `export`ed variables persist. If a variable prints blank later, you're in a new shell — re-run the `export` block. `expected one argument` after a flag means the variable you passed is empty (`echo $VAR` to check).

## Prerequisites (once per session)

```bash
az login                                                          # sign in (use --use-device-code from WSL if no browser opens)
az account show -o table                                          # confirm the active subscription
az extension add --name containerapp                              # adds the `az containerapp` commands
az extension add --name log-analytics                             # lets you query the environment's logs
az provider register --namespace Microsoft.App                    # Container Apps resource provider
az provider register --namespace Microsoft.OperationalInsights    # Log Analytics resource provider
az provider register --namespace Microsoft.ContainerRegistry      # ACR resource provider
```

---

# Lab 1 — Deploy a containerized backend API to Azure Container Apps

Folder: `mslearn-aca-deploy/` — a Flask **`ai-api`** mock document-processing app (gunicorn, port 8000). The starter `azdeploy.py` normally creates the ACR + builds the image + creates the environment; the commands below are that script decomposed.

### Variables (run once, same terminal)
```bash
export RESOURCE_GROUP=rg-ai200-aca-lab       # dedicated RG (one-line cleanup later)
export LOCATION=swedencentral                # region
export ACR_NAME=acrai200acalab               # globally-unique registry name
export ACR_SERVER=acrai200acalab.azurecr.io  # registry login server
export ACA_ENVIRONMENT=aca-ai200-lab-env     # Container Apps environment
export CONTAINER_APP_NAME=ai-api             # the app
export CONTAINER_IMAGE=ai-api:v1             # image:tag
export TARGET_PORT=8000                      # port gunicorn listens on
export MODEL_NAME=gpt-5.4-mini               # demo model name (plain env var)
export EMBEDDINGS_API_KEY=demo-key-12345     # demo key (stored as a secret)
```

### A. Infrastructure (what `azdeploy.py` does)
```bash
# create the resource group
az group create --name $RESOURCE_GROUP --location $LOCATION -o table

# Basic registry, admin DISABLED (we pull with a managed identity, not admin creds)
az acr create --resource-group $RESOURCE_GROUP --name $ACR_NAME --sku Basic --admin-enabled false -o table

# cloud build of ./api with ACR Tasks -> pushes ai-api:v1 (no local Docker needed)
az acr build --registry $ACR_NAME --image $CONTAINER_IMAGE ./api

# create the environment (auto-creates a Log Analytics workspace = the shared log sink)
az containerapp env create --name $ACA_ENVIRONMENT --resource-group $RESOURCE_GROUP --location $LOCATION -o table
```

### B. Deploy the app + wire up the secret
```bash
# create the app; --registry-identity system enables a system-assigned identity AND
# auto-assigns the AcrPull role on the registry (keyless pull, no admin password)
az containerapp create \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --environment $ACA_ENVIRONMENT \
  --image "$ACR_SERVER/$CONTAINER_IMAGE" \
  --ingress external \
  --target-port $TARGET_PORT \
  --env-vars MODEL_NAME=$MODEL_NAME \
  --registry-server "$ACR_SERVER" \
  --registry-identity system

# store the provider key as an app-level secret
az containerapp secret set -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP \
  --secrets embeddings-api-key=$EMBEDDINGS_API_KEY

# point an env var at the secret via secretref -> creates a NEW revision (restarts the app)
az containerapp update -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP \
  --set-env-vars EMBEDDINGS_API_KEY=secretref:embeddings-api-key

# confirm a second revision (--0000002) was created by the config change
az containerapp revision list -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP -o table
```

### C. Verify
```bash
# capture the public FQDN
FQDN=$(az containerapp show -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP \
  --query properties.configuration.ingress.fqdn -o tsv)
echo "$FQDN"

curl -s "https://$FQDN/health"     # -> {"status":"healthy"}  (ingress reaches the container on 8000)
curl -s "https://$FQDN/"           # -> model name + "embeddings_api_key_configured": true

echo "Azure Container Apps makes deploying containers simple." > document.txt
curl -s -X POST "https://$FQDN/process" -H "Content-Type: text/plain" -d @document.txt   # mock analysis JSON

az containerapp logs show -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP   # gunicorn startup + GET/POST access logs
```

### D. Best practice — move the secret to Azure Key Vault
Converts `embeddings-api-key` from a literal ACA value into a **Key Vault reference** (app code unchanged).
```bash
export KV_NAME=kv-ai200-$RANDOM     # run ONCE -> globally-unique vault name
echo "$KV_NAME"

# vault in Azure RBAC mode (so access is by role, which "Key Vault Secrets User" needs)
az keyvault create --name $KV_NAME --resource-group $RESOURCE_GROUP --location $LOCATION \
  --enable-rbac-authorization true -o table

# RBAC-mode gotcha: even a subscription Owner needs a data-plane role to write secret values
MY_ID=$(az ad signed-in-user show --query id -o tsv)
KV_ID=$(az keyvault show --name $KV_NAME --query id -o tsv)
az role assignment create --assignee "$MY_ID" --role "Key Vault Secrets Officer" --scope "$KV_ID"

# store the key (wait ~60s and retry if it says Forbidden while the role propagates)
az keyvault secret set --vault-name $KV_NAME --name embeddings-api-key --value "$EMBEDDINGS_API_KEY" -o table
export KV_SECRET_URI="https://$KV_NAME.vault.azure.net/secrets/embeddings-api-key"   # unversioned = latest (auto-rotation)

# grant the APP's managed identity read-only secret access
PRINCIPAL_ID=$(az containerapp identity show -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP --query principalId -o tsv)
az role assignment create --assignee "$PRINCIPAL_ID" --role "Key Vault Secrets User" --scope "$KV_ID"

# redefine the SAME secret name as a Key Vault reference using the app's system identity
az containerapp secret set -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP \
  --secrets "embeddings-api-key=keyvaultref:$KV_SECRET_URI,identityref:system"

# verify: the secret now shows a keyVaultUrl (no plaintext); the app still resolves it
az containerapp secret list -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP -o json
curl -s "https://$FQDN/"           # still "embeddings_api_key_configured": true, now sourced from Key Vault
```

### Cleanup
```bash
az group delete --name $RESOURCE_GROUP --no-wait --yes   # removes the RG and everything in it
az keyvault purge --name $KV_NAME                        # optional: Key Vault soft-delete retains the name ~90 days
```

### Takeaways
- The **environment** is a shared boundary (one VNet + one Log Analytics workspace); it auto-creates the workspace.
- `--registry-identity system` = keyless ACR pull; the CLI auto-creates the **AcrPull** role assignment for the app's managed identity.
- Secrets are **app-scoped**; changing config/secrets creates a **new revision**; an env var consumes a secret with `secretref:`.
- The container filesystem (`/tmp`) is **ephemeral** — lost on restart/scale; use Azure Files/Blob/Cosmos for persistence.
- Prod secret hygiene: don't keep literal values in ACA (the `listSecrets` RBAC action can return them in plain text). Prefer a **Key Vault reference** (`keyvaultref:` + identity), or better, a **managed identity** so there's no key at all.

---

# Lab 2 — Manage containers in Azure Container Apps

Folder: `mslearn-aca-manage/` (starter files download when I start this module).

_Coming when I do the module — updating images, managing revisions, health probes (liveness/readiness/startup), and troubleshooting with logs._

---

# Lab 3 — Scale containers in Azure Container Apps

Folder: `mslearn-aca-scale/` (starter files download when I start this module).

_Coming when I do the module — HTTP/TCP/CPU/memory scale rules, event-driven scaling with KEDA, and revision traffic splitting (blue/green, canary)._
