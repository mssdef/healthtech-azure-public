# infra/

Azure Bicep templates for healthtech-azure.

## Resources deployed

| Resource | Purpose |
|---|---|
| PostgreSQL Flexible Server (16) | Shared database for Phase 1 + Phase 2 |
| Azure Container Registry | Docker images for api-gateway |
| Azure Container Apps | NestJS api-gateway (autoscale 1–3) |
| Azure Static Web Apps | React patient-portal (free tier) |
| Azure Blob Storage | Case documents + EDI raw archive |
| Log Analytics Workspace | Container Apps logging |

## Deploy

```bash
# Login
az login
az account set --subscription <SUBSCRIPTION_ID>

# Create resource group
az group create --name healthtech-rg --location eastus

# Deploy
az deployment group create \
  --resource-group healthtech-rg \
  --template-file infra/main.bicep \
  --parameters infra/main.bicepparam \
  --parameters dbAdminPassword=$DB_ADMIN_PASSWORD
```

## Managed Identity

The api-gateway Container App uses system-assigned managed identity to pull from ACR.
No registry credentials are stored in environment variables.

## Secrets

Secrets (`DATABASE_URL`, `JWT_SECRET`) are injected via Container Apps secret references.
In production, source them from Azure Key Vault using managed identity.
