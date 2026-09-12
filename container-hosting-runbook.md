# Container Hosting Labs — Runbook (3 labs, in order)

All three labs from the MS Learn path **Implement container application hosting on Azure**, consolidated so you can do them first to last. Every command has a `#` comment explaining what it does. Starter files are in each `mslearn-*` folder.

> Keep one terminal per lab so the `export`ed variables persist. If a variable prints blank later, you're in a new shell, re-run the `export` block. Whenever you see `expected one argument` after a flag, the variable you passed is empty (`echo $VAR` to check).

## Prerequisites (once per session)

```bash
az login                                                       # sign in to Azure
az account show -o table                                       # confirm the active subscription
az provider register --namespace Microsoft.ContainerRegistry   # enable the ACR resource provider
az provider register --namespace Microsoft.Web                 # enable the App Service resource provider
```

---

# Lab 1 — Build and run a container image with ACR Tasks

Folder: `mslearn-acr-tasks/` (a Flask `inference-api` app).

### Setup
```bash
cd mslearn-acr-tasks              # into the ACR Tasks lab folder
export RG=rg-ai200-acr           # resource group that holds your registry
export ACR_NAME=acrai200demo88   # reuse your existing registry
```

### Build + verify
```bash
# build the image in Azure (ACR Tasks) from ./api and push it as inference-api:v1.0.0
az acr build --registry $ACR_NAME --image inference-api:v1.0.0 ./api
az acr repository list --name $ACR_NAME -o table                                  # list all repos in the registry
az acr repository show-tags --name $ACR_NAME --repository inference-api -o table   # list the repo's tags
az acr manifest list-metadata --registry $ACR_NAME --name inference-api -o table   # list images (manifests) + digests
```

### Run the image in the cloud (no local Docker)
```bash
# run the image in Azure and execute a command inside it (smoke test: import the Flask app)
az acr run --registry $ACR_NAME \
  --cmd "$ACR_NAME.azurecr.io/inference-api:v1.0.0 python -c 'from app import app'" /dev/null
```

### Second version, history, lock
```bash
az acr build --registry $ACR_NAME --image inference-api:v1.1.0 ./api               # build a second version tag
az acr repository show-tags --name $ACR_NAME --repository inference-api -o table    # confirm both tags exist
az acr task list-runs --registry $ACR_NAME -o table                                # history of every ACR Tasks run
# lock v1.0.0 so it can't be overwritten or deleted
az acr repository update --name $ACR_NAME --image inference-api:v1.0.0 --write-enabled false
az acr repository show --name $ACR_NAME --image inference-api:v1.0.0                # verify (writeEnabled=false)
```

### Cleanup (keep the registry, remove just this repo)
```bash
# re-enable write so the locked image can be removed
az acr repository update --name $ACR_NAME --image inference-api:v1.0.0 --write-enabled true
az acr repository delete --name $ACR_NAME --repository inference-api --yes          # delete the whole repo
```

---

# Lab 2 — Deploy a container to Azure App Service

Folder: `mslearn-app-svc-container/` (a Flask `docprocessor`, gunicorn on port 80).

### Setup + build
```bash
cd ../mslearn-app-svc-container            # into the App Service container lab folder
export RESOURCE_GROUP=rg-ai200-acr         # resource group (reuses the registry's RG)
export ACR_NAME=acrai200demo88             # your registry
export LOCATION=swedencentral              # region
export APP_PLAN=plan-docprocessor          # App Service plan name
export APP_NAME=app-docprocessor-$RANDOM   # globally-unique web app name
echo "APP_NAME=$APP_NAME"                  # print the generated name (note it down)

az acr build -r $ACR_NAME -t docprocessor:v1 ./api                                  # build the image in the cloud
az appservice plan create -g $RESOURCE_GROUP -n $APP_PLAN --is-linux --sku B1 -l $LOCATION  # Linux Basic B1 plan
```

### Web app + keyless pull
```bash
# create the web app pointing at your private ACR image
az webapp create -g $RESOURCE_GROUP -p $APP_PLAN -n $APP_NAME \
  --container-image-name $ACR_NAME.azurecr.io/docprocessor:v1
az webapp identity assign -g $RESOURCE_GROUP -n $APP_NAME                           # turn on a system-assigned identity
PRINCIPAL_ID=$(az webapp identity show -g $RESOURCE_GROUP -n $APP_NAME --query principalId -o tsv)  # its object ID
ACR_ID=$(az acr show -g $RESOURCE_GROUP -n $ACR_NAME --query id -o tsv)             # the registry's resource ID (role scope)
# grant that identity AcrPull on the registry (pull-only, least privilege)
az role assignment create --assignee-object-id $PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal --role AcrPull --scope $ACR_ID
# tell App Service to pull using the system-assigned identity, not a password
az webapp config set -g $RESOURCE_GROUP -n $APP_NAME --acr-use-identity true --acr-identity "[system]"
```

### Runtime settings, verify, test
```bash
# container port (80) + enable the persistent /home volume
az webapp config appsettings set -g $RESOURCE_GROUP -n $APP_NAME \
  --settings WEBSITES_PORT=80 WEBSITES_ENABLE_APP_SERVICE_STORAGE=true
az webapp config set -g $RESOURCE_GROUP -n $APP_NAME --always-on true               # keep it warm (no cold starts)
az webapp log config -g $RESOURCE_GROUP -n $APP_NAME --docker-container-logging filesystem  # capture stdout/stderr
az webapp restart -g $RESOURCE_GROUP -n $APP_NAME                                   # restart to pull with the identity

APP_URL=$(az webapp show -g $RESOURCE_GROUP -n $APP_NAME --query defaultHostName -o tsv)  # the public hostname
sleep 45                                                                            # wait for first cold pull + start
curl -s https://$APP_URL/                                                           # root: service info JSON
# POST the sample document to the processing endpoint
curl -X POST "https://$APP_URL/process" -H "Content-Type: text/plain" --data-binary @document.txt
curl -s https://$APP_URL/documents                                                  # read docs back from persistent storage
```
> If the pull fails "forbidden/unauthorized", AcrPull is still propagating, wait a minute and `az webapp restart`.

### Cleanup (keep the registry)
```bash
az webapp delete -g $RESOURCE_GROUP -n $APP_NAME                                    # delete the web app
az appservice plan delete -g $RESOURCE_GROUP -n $APP_PLAN --yes                     # delete the plan
az acr repository delete -n $ACR_NAME --repository docprocessor --yes              # remove the image (registry stays)
```

---

# Lab 3 — Deploy an AI API with a local model-serving sidecar

Folder: `mslearn-app-svc-sidecar/` (chat-api main + Phi-3 model-server sidecar).
> COST: uses a **P2v3** plan (~$0.40/hr). Do it in one sitting and run the cleanup promptly.

### Setup
```bash
cd ../mslearn-app-svc-sidecar              # into the sidecar lab folder
export RESOURCE_GROUP=rg-sidecar-lab       # dedicated RG (the P2v3 plan lives here)
export LOCATION=swedencentral              # region
export ACR_NAME=acrsidecar$RANDOM          # new, globally-unique registry name
export IDENTITY_NAME=id-sidecar            # user-assigned identity name
export APP_PLAN=plan-sidecar               # App Service plan name
export APP_NAME=app-ai-sidecar-$RANDOM     # globally-unique web app name
az group create -n $RESOURCE_GROUP -l $LOCATION   # create the resource group
```

### 1) Registry + both images
```bash
# create the registry (admin disabled; we authenticate with managed identity)
az acr create -g $RESOURCE_GROUP -n $ACR_NAME -l $LOCATION --sku Basic --admin-enabled false
# confirm ARM-auth policy is "enabled" (required for managed-identity pulls)
az acr config authentication-as-arm show --registry $ACR_NAME -g $RESOURCE_GROUP --query status -o tsv
az acr build -r $ACR_NAME -t chat-api:v1 ./api                    # build the main chat API image
az acr build -r $ACR_NAME -t model-server:v1 ./model-server       # build Phi-3 image (~2.7GB, 5-10 min)
```

### 2) User-assigned identity + AcrPull
```bash
az identity create -g $RESOURCE_GROUP -n $IDENTITY_NAME -l $LOCATION                 # create the user-assigned identity
IDENTITY_RESOURCE_ID=$(az identity show -g $RESOURCE_GROUP -n $IDENTITY_NAME --query id -o tsv)          # attach it to the app later
IDENTITY_CLIENT_ID=$(az identity show -g $RESOURCE_GROUP -n $IDENTITY_NAME --query clientId -o tsv)      # goes into the spec
IDENTITY_PRINCIPAL_ID=$(az identity show -g $RESOURCE_GROUP -n $IDENTITY_NAME --query principalId -o tsv) # for the role assignment
ACR_ID=$(az acr show -g $RESOURCE_GROUP -n $ACR_NAME --query id -o tsv)              # registry resource ID (role scope)
# grant the identity AcrPull on the registry
az role assignment create --assignee-object-id $IDENTITY_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal --role AcrPull --scope $ACR_ID
```

### 3) P2v3 plan + sidecar web app + generate the spec
```bash
az appservice plan create -g $RESOURCE_GROUP -n $APP_PLAN -l $LOCATION --sku P2v3 --is-linux  # Premium v3 (model memory)
az webapp create -g $RESOURCE_GROUP -p $APP_PLAN -n $APP_NAME --sitecontainers-app            # sidecar-enabled web app
az webapp identity assign -g $RESOURCE_GROUP -n $APP_NAME --identities $IDENTITY_RESOURCE_ID  # attach the user-assigned identity
az webapp config set -g $RESOURCE_GROUP -n $APP_NAME --generic-configurations '{"acrUseManagedIdentityCreds": true}'  # pull with MI
az webapp config set -g $RESOURCE_GROUP -n $APP_NAME --always-on true                         # keep it warm
# fill the spec template with your registry name + identity client ID
sed -e "s|<registry-name>|$ACR_NAME|g" -e "s|<managed-identity-client-id>|$IDENTITY_CLIENT_ID|g" \
  sitecontainers-spec.template.json > sitecontainers-spec.json
cat sitecontainers-spec.json                                                                 # review the generated spec
```

### 4) Apply the spec + verify
```bash
export CHAT_API_URL="https://$APP_NAME.azurewebsites.net"                            # the public API URL
# define the main + sidecar containers from the spec (starts the image pulls)
az webapp sitecontainers create -g $RESOURCE_GROUP -n $APP_NAME --sitecontainers-spec-file ./sitecontainers-spec.json
az webapp sitecontainers list -g $RESOURCE_GROUP -n $APP_NAME -o table               # confirm both containers (roles + ports)
# watch the model load; Ctrl+C after "listening on 11434"
az webapp sitecontainers log -g $RESOURCE_GROUP -n $APP_NAME --container-name model-server
curl --fail-with-body "$CHAT_API_URL/health/ready"   # main API reaches the sidecar over localhost:11434
curl --fail-with-body "$CHAT_API_URL/model-info"     # main API reads the manifest from the shared /home volume
```

### 5) Local chat client (optional)
```bash
cd client                                                                            # into the client app
python3 -m venv ~/sidecar-client-venv && source ~/sidecar-client-venv/bin/activate   # venv on ~ (NOT /mnt/c)
pip install -r requirements.txt                                                      # install flask + requests
export CHAT_API_URL="https://$APP_NAME.azurewebsites.net" CHAT_API_TIMEOUT=300        # point the client at the API
python app.py                                                                        # serve at http://127.0.0.1:5000
```
> Hosting the client as a container + requiring login (Easy Auth): see [client-hosting.md](mslearn-app-svc-sidecar/client-hosting.md).

### Cleanup (the whole resource group)
```bash
az group delete --name rg-sidecar-lab --no-wait --yes   # deletes registry, plan, both apps, and the identity
```

---

## Quick reference: what each lab proves

| Lab | Core skill | Signature commands |
|---|---|---|
| 1. ACR Tasks | Build/run/version images in the cloud | `az acr build`, `az acr run`, `--write-enabled false` |
| 2. App Service | Keyless container hosting | `az webapp create`, identity + `AcrPull`, `WEBSITES_PORT` |
| 3. Sidecar | Multi-container AI app | `--sitecontainers-app`, `sitecontainers create`, `localhost` + `/home` |
