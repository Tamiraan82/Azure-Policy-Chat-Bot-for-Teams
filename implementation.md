# Implementation Specification: Azure Policy Chat Bot

This document outlines the infrastructure, identity, and deployment requirements for the Azure Policy Chat Bot, following a **Zero-Secret** and **Federated Identity** model.

## 1. Prerequisites & Versioning

To ensure compatibility and security, use the following versions:

| Category | Requirement | Version |
| :--- | :--- | :--- |
| **Infrastructure** | Terraform | `~> 1.6.0` |
| | Azure RM Provider | `~> 3.90.0` |
| **Logic** | Python | `3.12-slim` |
| | Bot Framework SDK | `~> 4.15.0` |
| **CLI** | Azure CLI | `~> 2.56.0` |

## 2. Azure Regional Strategy

Deploy all resources in a single region where **Azure OpenAI** is available to minimize latency.

> [!IMPORTANT]
> **Recommended Region**: `swedencentral` or `eastus2`.
> Sweden Central offers superior availability for GPT-4 (including Turbo).

## 3. Required Azure Resources

| Resource | Service | Purpose |
| :--- | :--- | :--- |
| **Compute** | Azure Container Apps | Hosts the Python bot backend with User-Assigned Identity. |
| **Intelligence** | Azure OpenAI Service | Intent extraction and KQL generation (**GPT-4o** or **GPT-4 Turbo**). |
| **Security** | Azure Key Vault | Stores application configuration (No client secrets). |
| **Identity** | User-Assigned Managed Identity | Zero-secret identity for the Container App. |
| **Storage** | Azure Container Registry | Stores the bot's Docker images. |
| **Monitoring** | Application Insights | Logging, telemetry, and performance tracking. |

## 4. Federated Identity & Zero-Secret Strategy

This solution eliminates long-lived client secrets in favor of **OIDC Federation** and **Managed Identities**.

### A. Infrastructure Deployment (Terraform via ADO)
Instead of using a Service Principal with a secret, use **Workload Identity Federation**:
1. Create an Azure DevOps Service Connection using **Workload Identity Federation**.
2. Configure the Terraform provider to use OIDC:
   ```hcl
   provider "azurerm" {
     features {}
     use_oidc = true
   }
   ```

### B. Bot App Registration (Federated Credentials)
The Bot App Registration (used for Teams) communicates with the backend via **Federated Credentials**:
- **Subject**: The User-Assigned Managed Identity of the Container App.
- **Trust**: The Bot App registration trusts the Managed Identity, eliminating the need for a `MICROSOFT_APP_PASSWORD`.

### C. API / OBO App Registration
- **Purpose**: Facilitates On-Behalf-Of (OBO) token exchange for Azure Graph/ARM.
- **Federation**: This app should also be configured with a Federated Credential trust for the Bot's Managed Identity.
- **Permissions**: `Azure Service Management (user_impersonation)`.

## 5. OpenAI Model Selection for KQL

Generating accurate Kusto Query Language (KQL) for Azure Resource Graph requires high reasoning capabilities and a large context window to process complex policy schemas.

| Model | Recommended Version | Strengths |
| :--- | :--- | :--- |
| **GPT-4o** | `2024-05-13` (or later) | **Primary Choice**. Superior speed and reasoning for KQL syntax. |
| **GPT-4 Turbo** | `2024-04-09` | High stability and large context window (128k) for multi-shot prompting. |

> [!TIP]
> **Prompting Strategy**: Provide the LLM with a schema snippet of `Microsoft.ResourceGraph/resources` to improve KQL accuracy for exotic resource types.

## 6. Roles & Identities (Least Privilege)

The Bot backend has **zero** standing reader/contributor roles on subscriptions. All compliance data is fetched using the user's delegated identity.

### Managed Identity Roles (Bot Backend)
- `Key Vault Secrets User`: Required to fetch app configuration.
- `Cognitive Services User`: Required to call Azure OpenAI.
- `AcrPull`: For the Container App to pull its own image.

### User RBAC Requirements
- Users must have `Reader` permissions on the target subscriptions/resource groups to see compliance data.

## 7. Architecture Diagrams

### System Overview
![Technical Architecture](docs/images/architecture.png)

### Authentication Flow (OBO)
![OAuth2 OBO Flow](docs/images/auth_flow.png)

## 8. Terraform Implementation Snippet (OIDC & Managed Identity)

```hcl
# Use Managed Identity for the Container App
resource "azurerm_user_assigned_identity" "bot" {
  name                = "id-policy-bot"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
}

# Container App with zero secrets in ENV
resource "azurerm_container_app" "bot" {
  name                         = "ca-policy-bot"
  resource_group_name          = azurerm_resource_group.main.name
  container_app_environment_id = azurerm_container_app_environment.main.id
  revision_mode                = "Single"

  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.bot.id]
  }

  template {
    container {
      name   = "bot-backend"
      image  = "botregistry.azurecr.io/policy-bot:latest"
      cpu    = 0.5
      memory = "1Gi"
      
      env {
        name  = "AZURE_CLIENT_ID"
        value = azurerm_user_assigned_identity.bot.client_id
      }
      env {
        name  = "AZURE_TENANT_ID"
        value = data.azurerm_client_config.current.tenant_id
      }
    }
  }
}
```
