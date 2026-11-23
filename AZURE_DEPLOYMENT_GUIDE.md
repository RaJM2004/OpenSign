# Azure DevOps Deployment Guide for OpenSign (GxP / 21 CFR Part 11 Ready)

This guide provides step-by-step instructions for DevOps and Cloud Platform engineers to deploy OpenSign (Frontend, Backend, and MongoDB) onto **Microsoft Azure** using enterprise best practices, high availability, and compliance with **FDA 21 CFR Part 11 / GxP**.

---

## 1. Architecture Overview

```
                                      +------------------------------------+
                                      |          End Users / Signers       |
                                      +-----------------+------------------+
                                                        |
                                                        | HTTPS (443)
                                                        v
                                      +------------------------------------+
                                      |    Azure Front Door / App Gateway  |
                                      |  (WAF, Custom Domain, SSL/TLS Offload)
                                      +--------+-------------------+-------+
                                               |                   |
                        Route: /api/* (API)     |                   | Route: /* (UI)
                                               v                   v
                        +----------------------+---+       +-------+-------------------+
                        | OpenSign Backend Server  |       | OpenSign Frontend Client  |
                        | (Azure Container Apps)   |       | (Azure Container Apps)    |
                        | Node.js 22 + LibreOffice |       | Vite / React SPA (Nginx)  |
                        +-----------+--------------+       +---------------------------+
                                    |
            +-----------------------+-----------------------+
            |                       |                       |
            v                       v                       v
+-----------+-----------+ +---------+-----------+ +---------+-----------+
| Azure Cosmos DB for   | | Azure Files Share   | | Azure Key Vault     |
| MongoDB (vCore)       | | /usr/src/app/files  | | PFX Cert, Passphrase|
| (Persistent & Backed) | | (Document Storage)  | | Master Key, Secrets |
+-----------------------+ +---------------------+ +---------------------+
```

### Components
1. **Frontend (`apps/OpenSign`)**: React/Vite client served on port `3000`.
2. **Backend Server (`apps/OpenSignServer`)**: Node.js 22 Parse-Server & Express API on port `8080`, with LibreOffice for document conversion and pdf-lib for digital signing (`.pfx`) & GxP audit trail attachment.
3. **Database**: Azure Cosmos DB for MongoDB (vCore cluster) or MongoDB Atlas on Azure.
4. **Storage**: Azure Files (SMB/NFS) mounted to `/usr/src/app/files` or S3-compatible Azure Blob Storage.
5. **Key Management**: Azure Key Vault for digital signature certificate (`PFX_BASE64`), passphrases, and master keys.

---

## 2. Prerequisites

- **Azure CLI** installed (`az login`)
- **Docker** and **Git** installed locally or on CI/CD runner
- Azure Subscription with permissions to create Resource Groups, Container Registries (ACR), Container Apps, and Cosmos DB.

---

## 3. Azure Resource Provisioning

### Step 1: Set Variables & Create Resource Group
```bash
export RESOURCE_GROUP="rg-opensign-prod"
export LOCATION="eastus" # or westeurope, centralus, etc.
export ACR_NAME="acropensignprod"
export ENVIRONMENT_NAME="cae-opensign-prod"
export STORAGE_ACCOUNT="saopensignprod"
export FILE_SHARE_NAME="opensign-documents"
export COSMOS_CLUSTER="cosmos-opensign-prod"

# Create Resource Group
az group create --name $RESOURCE_GROUP --location $LOCATION
```

### Step 2: Create Azure Container Registry (ACR)
```bash
az acr create \
  --resource-group $RESOURCE_GROUP \
  --name $ACR_NAME \
  --sku Standard \
  --admin-enabled true

# Retrieve credentials
ACR_LOGIN_SERVER=$(az acr show --name $ACR_NAME --query loginServer --output tsv)
ACR_PASSWORD=$(az acr credential show --name $ACR_NAME --query "passwords[0].value" --output tsv)
```

### Step 3: Provision MongoDB (Azure Cosmos DB for MongoDB vCore)
```bash
# Recommended for production: vCore cluster with high throughput & native indexing
az cosmosdb mongocluster create \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $COSMOS_CLUSTER \
  --location $LOCATION \
  --administrator-login "opensignadmin" \
  --administrator-login-password "YourStrongPassword#2026" \
  --node-count 1 \
  --node-size "M30" \
  --tier "Burstable"

# Retrieve MongoDB Connection String:
# mongodb+srv://opensignadmin:YourStrongPassword%232026@cosmos-opensign-prod.mongocluster.cosmos.azure.com/?tls=true&authMechanism=SCRAM-SHA-256&retrywrites=false&maxIdleTimeMS=120000
```
> *Alternative: You may also use an existing MongoDB Atlas on Azure or a self-hosted MongoDB VM / Container.*

### Step 4: Storage Account & Azure File Share (for Persistent PDF storage)
```bash
# Create storage account
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2

# Get Storage Account Key
STORAGE_KEY=$(az storage account keys list --resource-group $RESOURCE_GROUP --account-name $STORAGE_ACCOUNT --query "[0].value" --output tsv)

# Create File Share for documents
az storage share create \
  --name $FILE_SHARE_NAME \
  --account-name $STORAGE_ACCOUNT \
  --account-key $STORAGE_KEY \
  --quota 100
```

---

## 4. Building & Pushing Container Images to ACR

Run these commands from the root of the OpenSign repository:

```bash
# Log in to ACR
az acr login --name $ACR_NAME

# 1. Build and push Backend image (includes LibreOffice for DOCX conversions)
docker build -t ${ACR_LOGIN_SERVER}/opensignserver:latest -f apps/OpenSignServer/Dockerhubfile .
docker push ${ACR_LOGIN_SERVER}/opensignserver:latest

# 2. Build and push Frontend image
docker build -t ${ACR_LOGIN_SERVER}/opensign:latest -f apps/OpenSign/Dockerhubfile .
docker push ${ACR_LOGIN_SERVER}/opensign:latest
```

---

## 5. Deploying on Azure Container Apps (Recommended)

Azure Container Apps (ACA) provides serverless container execution with automated TLS, microservices ingress, and native support for Azure Files volume mounts.

### Step 1: Create Container Apps Environment
```bash
az containerapp env create \
  --name $ENVIRONMENT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

# Link Azure File Share to the Environment
az containerapp env storage set \
  --name $ENVIRONMENT_NAME \
  --resource-group $RESOURCE_GROUP \
  --storage-name "docfileshare" \
  --azure-file-account-name $STORAGE_ACCOUNT \
  --azure-file-account-key $STORAGE_KEY \
  --azure-file-share-name $FILE_SHARE_NAME \
  --access-mode ReadWrite
```

### Step 2: Deploy OpenSign Backend Server
Create `backend-app.json` or run CLI:

```bash
az containerapp create \
  --name "opensign-backend" \
  --resource-group $RESOURCE_GROUP \
  --environment $ENVIRONMENT_NAME \
  --image "${ACR_LOGIN_SERVER}/opensignserver:latest" \
  --registry-server $ACR_LOGIN_SERVER \
  --registry-username $ACR_NAME \
  --registry-password $ACR_PASSWORD \
  --target-port 8080 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 5 \
  --cpu 1.0 \
  --memory 2.0Gi \
  --env-vars \
    NODE_ENV="production" \
    PORT="8080" \
    APP_ID="opensign" \
    MASTER_KEY="<GENERATE_SECURE_RANDOM_UUID>" \
    SERVER_URL="https://<BACKEND_INGRESS_URL>/app" \
    PUBLIC_URL="https://<FRONTEND_INGRESS_URL>" \
    MONGODB_URI="mongodb+srv://opensignadmin:YourStrongPassword%232026@cosmos-opensign-prod.mongocluster.cosmos.azure.com/opensign?tls=true&authMechanism=SCRAM-SHA-256" \
    PARSE_MOUNT="/app" \
    SMTP_ENABLE="true" \
    SMTP_HOST="smtp-relay.brevo.com" \
    SMTP_PORT="587" \
    SMTP_USERNAME="your-smtp-account" \
    SMTP_PASS="your-smtp-password" \
    SMTP_USER_EMAIL="no-reply@yourdomain.com" \
    appName="OpenSign™" \
    PFX_BASE64="<YOUR_BASE64_ENCODED_PFX_CERTIFICATE>" \
    PASS_PHRASE="<YOUR_PFX_PASSWORD>"

# Mount Azure File Share to /usr/src/app/files inside Backend container:
az containerapp volume mount \
  --name "opensign-backend" \
  --resource-group $RESOURCE_GROUP \
  --volume-name "docfileshare" \
  --mount-path "/usr/src/app/files"
```

### Step 3: Deploy OpenSign Frontend
```bash
BACKEND_FQDN=$(az containerapp show --name "opensign-backend" --resource-group $RESOURCE_GROUP --query "properties.configuration.ingress.fqdn" --output tsv)

az containerapp create \
  --name "opensign-frontend" \
  --resource-group $RESOURCE_GROUP \
  --environment $ENVIRONMENT_NAME \
  --image "${ACR_LOGIN_SERVER}/opensign:latest" \
  --registry-server $ACR_LOGIN_SERVER \
  --registry-username $ACR_NAME \
  --registry-password $ACR_PASSWORD \
  --target-port 3000 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 5 \
  --cpu 0.5 \
  --memory 1.0Gi \
  --env-vars \
    REACT_APP_SERVERURL="https://${BACKEND_FQDN}/app" \
    REACT_APP_APPID="opensign" \
    PUBLIC_URL="https://<FRONTEND_FQDN>"
```

---

## 6. Alternative: Multi-Container Deployment via Azure Web App (Docker Compose)

For simple or single-VM environments, you can use **Azure App Service for Linux (Web App for Containers)** with Docker Compose.

1. Create a `docker-compose.azure.yml`:
```yaml
version: '3.8'

services:
  backend:
    image: <ACR_NAME>.azurecr.io/opensignserver:latest
    container_name: opensign-backend
    ports:
      - "8080:8080"
    environment:
      - NODE_ENV=production
      - PORT=8080
      - APP_ID=opensign
      - MASTER_KEY=${MASTER_KEY}
      - SERVER_URL=https://${CUSTOM_DOMAIN}/api/app
      - PUBLIC_URL=https://${CUSTOM_DOMAIN}
      - MONGODB_URI=${MONGODB_URI}
      - SMTP_ENABLE=true
      - SMTP_HOST=${SMTP_HOST}
      - SMTP_PORT=587
      - SMTP_USERNAME=${SMTP_USERNAME}
      - SMTP_PASS=${SMTP_PASS}
      - SMTP_USER_EMAIL=${SMTP_USER_EMAIL}
      - PFX_BASE64=${PFX_BASE64}
      - PASS_PHRASE=${PASS_PHRASE}
    volumes:
      - docshare:/usr/src/app/files

  frontend:
    image: <ACR_NAME>.azurecr.io/opensign:latest
    container_name: opensign-frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_SERVERURL=https://${CUSTOM_DOMAIN}/api/app
      - REACT_APP_APPID=opensign
      - PUBLIC_URL=https://${CUSTOM_DOMAIN}

  caddy:
    image: caddy:latest
    container_name: caddy-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile

volumes:
  docshare:
```

2. Deploy using Azure CLI:
```bash
az webapp create \
  --resource-group $RESOURCE_GROUP \
  --plan "asp-opensign-prod" \
  --name "app-opensign-prod" \
  --multicontainer-config-type COMPOSE \
  --multicontainer-config-file docker-compose.azure.yml
```

---

## 7. GxP / FDA 21 CFR Part 11 Compliance Configuration Checklist

For life sciences, pharmaceutical, and healthcare deployments requiring **FDA 21 CFR Part 11 compliance**:

| Requirement | Azure / OpenSign Configuration |
| :--- | :--- |
| **Audit Trail Retention** | Audit trails are permanently stored in MongoDB and attached directly to the final completed signed PDF. Enable continuous backups on Cosmos DB with point-in-time recovery (PITR). |
| **Secure Digital Certificate** | Store your organization's `.pfx` digital certificate and password in **Azure Key Vault**. Inject them into container environment variables at runtime via Managed Identity (`@Microsoft.KeyVault(SecretUri=...)`). |
| **Tamper-Evident Storage** | Store archived signed PDFs in an **Azure Blob Storage container with Immutable Blob Storage (WORM policy - Write Once, Read Many)**. |
| **UTC Timestamping & NTP** | All Azure container host nodes synchronize time via Azure's internal stratum 1 NTP clocks. All audit events log ISO 8601 UTC timestamps. |
| **Electronic Signature Reasons** | Signers must select declared signing reasons (*"Author"*, *"Approver"*, *"Reviewer"*), re-authenticate, and agree to the 21 CFR Part 11 attestation modal before applying signatures. |
| **Access Control & RBAC** | Restrict Azure portal & database access using Azure Entra ID (formerly Azure AD) with Privileged Identity Management (PIM) and MFA. |

---

## 8. Automated CI/CD Pipeline (GitHub Actions)

Add this workflow to `.github/workflows/deploy-azure.yml`:

```yaml
name: Deploy OpenSign to Azure

on:
  push:
    branches:
      - main
      - staging

env:
  AZURE_CONTAINER_APP_NAME_BACKEND: opensign-backend
  AZURE_CONTAINER_APP_NAME_FRONTEND: opensign-frontend
  RESOURCE_GROUP: rg-opensign-prod

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Log in to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Log in to Azure Container Registry (ACR)
        uses: azure/docker-login@v1
        with:
          login-server: ${{ secrets.ACR_LOGIN_SERVER }}
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      - name: Build & Push Backend
        run: |
          docker build -t ${{ secrets.ACR_LOGIN_SERVER }}/opensignserver:${{ github.sha }} -t ${{ secrets.ACR_LOGIN_SERVER }}/opensignserver:latest -f apps/OpenSignServer/Dockerhubfile .
          docker push ${{ secrets.ACR_LOGIN_SERVER }}/opensignserver:${{ github.sha }}
          docker push ${{ secrets.ACR_LOGIN_SERVER }}/opensignserver:latest

      - name: Build & Push Frontend
        run: |
          docker build -t ${{ secrets.ACR_LOGIN_SERVER }}/opensign:${{ github.sha }} -t ${{ secrets.ACR_LOGIN_SERVER }}/opensign:latest -f apps/OpenSign/Dockerhubfile .
          docker push ${{ secrets.ACR_LOGIN_SERVER }}/opensign:${{ github.sha }}
          docker push ${{ secrets.ACR_LOGIN_SERVER }}/opensign:latest

      - name: Update Azure Container Apps
        uses: azure/CLI@v1
        with:
          inlineScript: |
            az containerapp update \
              --name $AZURE_CONTAINER_APP_NAME_BACKEND \
              --resource-group $RESOURCE_GROUP \
              --image ${{ secrets.ACR_LOGIN_SERVER }}/opensignserver:${{ github.sha }}
            
            az containerapp update \
              --name $AZURE_CONTAINER_APP_NAME_FRONTEND \
              --resource-group $RESOURCE_GROUP \
              --image ${{ secrets.ACR_LOGIN_SERVER }}/opensign:${{ github.sha }}
```

---

## 9. Verification & Post-Deployment Testing

1. **Verify Backend Health**:
   ```bash
   curl -I https://<BACKEND_INGRESS_URL>/app
   # Should return HTTP 403 (Forbidden by Parse Server without App ID, confirming API is live)
   ```
2. **Verify Frontend Access**:
   - Open `https://<FRONTEND_INGRESS_URL>` in your browser.
   - On the first deployment with a fresh database, navigate to `/addadmin` to initialize the super administrator user.
3. **Verify DOCX to PDF Conversion**:
   - Upload a test `.docx` document to ensure LibreOffice executes properly inside the container without missing library errors.
4. **Verify GxP Audit Trail Attachment**:
   - Create a document with **GxP (FDA 21 CFR Part 11)** mode enabled.
   - Sign the document via the recipient link, select a reason (e.g. *I am the author of this document*), and confirm.
   - Download the completed PDF and confirm that the multi-page Audit Trail Certificate is attached directly to the end of the document.
