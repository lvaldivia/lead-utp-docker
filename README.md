# Workshop: Containers on Azure — From Docker to Azure Container Apps

Step-by-step guide for the full flow: local Docker container → Azure Container Registry (ACR) → Azure Container Apps.

## Prerequisites

- **Azure CLI** installed
  ```powershell
  winget install -e --id Microsoft.AzureCLI
  ```
  Verify installation:
  ```powershell
  az --version
  ```
- **Docker** installed and running (Docker Desktop on Windows), if you want to build locally instead of building remotely on Azure.
- An **Azure account with an active subscription**. If it's a new account, activate one at https://azure.microsoft.com/free/ or check if you qualify for **Azure for Students**.
- Account with **MFA configured**: https://aka.ms/mfasetup

## 1. Project structure (Python Hello World)

```
docker/
├── app.py
├── requirements.txt
├── Dockerfile
└── .dockerignore
```

### `app.py`

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### `requirements.txt`

```
flask
```

### `Dockerfile`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN useradd -m appuser
USER appuser

EXPOSE 5000

CMD ["python", "app.py"]
```

### `.dockerignore`

```
__pycache__/
*.pyc
.git
.env
```

## 2. Local build and test (optional, if Docker is running)

```powershell
docker build -t hello-world .
docker run -p 5000:5000 hello-world
```

Test it: open `http://localhost:5000` or `curl http://localhost:5000`.

Cleanup:
```powershell
docker rm -f $(docker ps -aq --filter ancestor=hello-world)
docker rmi hello-world
```

## 3. Log in to Azure

```powershell
az login
```

Verify the active subscription:
```powershell
az account list --output table
```

### Troubleshooting the login

If `az login` doesn't show your subscription, or you get an error, try these in order:

**"No subscriptions found" / empty `az account list`**
```powershell
az account clear
az login
```
This clears cached (possibly stale) credentials and forces a fresh interactive login.

**`AADSTS50076` — MFA required**
Set up multi-factor authentication at https://aka.ms/mfasetup with the account you're using, then retry `az login`, making sure to complete the MFA step in the browser (don't let it close early).

**Still not finding your subscription — force your own tenant**
Every Azure account belongs to a tenant (organization), and each person's tenant ID is different — never reuse someone else's. Find yours either from the error message Azure CLI shows, or from the Azure Portal under **Azure Active Directory → Overview → Tenant ID**. Then log in explicitly against it:
```powershell
az login --tenant YOUR_TENANT_ID
```

## 4. Register the required resource providers

On a new subscription, these namespaces aren't enabled by default:

```powershell
az provider register --namespace Microsoft.ContainerRegistry
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.ContainerInstance
```

Check the status (repeat until it shows `"Registered"`):
```powershell
az provider show --namespace Microsoft.ContainerRegistry --query registrationState
```

## 5. Create the Resource Group

```powershell
az group create --name rg-workshop-azure --location eastus
```

## 6. Create the Azure Container Registry (ACR)

```powershell
az acr create --resource-group rg-workshop-azure --name acrworkshopazure --sku Basic
```

> ACR names are globally unique across all of Azure — if it's taken, use another one (e.g. `acrworkshopazureyourname123`).

Enable the admin user (needed to push/pull with username and password):

```powershell
az acr update --name acrworkshopazure --admin-enabled true
```

## 7. Build the image directly on Azure (no local Docker needed)

From the folder containing the `Dockerfile`:

```powershell
az acr build --registry acrworkshopazure --image hello-world:v1 .
```

Alternative with local Docker (build + manual push):
```powershell
az acr login --name acrworkshopazure
docker tag hello-world acrworkshopazure.azurecr.io/hello-world:v1
docker push acrworkshopazure.azurecr.io/hello-world:v1
```

## 8. Get the ACR credentials (without exposing them on screen)

```powershell
$acrUsername = az acr credential show --name acrworkshopazure --query "username" --output tsv
$acrPassword = az acr credential show --name acrworkshopazure --query "passwords[0].value" --output tsv
```

## 9. Deploy to Azure Container Apps

```powershell
az containerapp up `
  --name hello-world-app `
  --resource-group rg-workshop-azure `
  --location eastus2 `
  --environment workshop-env `
  --image acrworkshopazure.azurecr.io/hello-world:v1 `
  --target-port 5000 `
  --ingress external `
  --registry-server acrworkshopazure.azurecr.io `
  --registry-username $acrUsername `
  --registry-password $acrPassword
```

> If you get `AKSCapacityHeavyUsage` in a region, try another one (`eastus2`, `centralus`, `westus2`). The Container App doesn't need to be in the same region as the Resource Group or the ACR.
>
> If the environment already exists in a different region, delete it before retrying in the new one:
> ```powershell
> az containerapp env delete --name workshop-env --resource-group rg-workshop-azure --yes
> ```
> (takes 2–5 minutes)

Once it finishes, the command returns a public URL like:
```
https://hello-world-app.xxxxx.eastus2.azurecontainerapps.io
```

That URL already comes with **automatic TLS** (HTTPS).

## 10. Scale the app (optional, bonus)

```powershell
az containerapp update `
  --name hello-world-app `
  --resource-group rg-workshop-azure `
  --min-replicas 1 --max-replicas 5
```

## 11. Security — rotate credentials after the workshop

If the ACR password was shown on screen, in chat, or in a recording during the demo, invalidate it afterward:

```powershell
az acr credential renew --name acrworkshopazure --password-name password
```

## Flow summary

```
Dockerfile + app.py
        │
        ▼
 az acr build  ──────────►  Azure Container Registry (ACR)
        │
        ▼
 az containerapp up  ────►  Azure Container Apps  ────►  Public HTTPS URL
```