# Azure Policy Chat Bot for Teams

An intelligent governance bot designed to answer compliance questions using natural language. It leverages Azure OpenAI (GPT-4o) to translate user intent into Kusto Query Language (KQL) for Azure Resource Graph.

## 🚀 Project Overview

This bot acts as a stateless, zero-trust proxy between Microsoft Teams and Azure Management APIs. It ensures that data retrieval is always performant, secure, and filtered by the user's specific RBAC permissions.

---

## 🏗️ Platform Analysis: Why Custom?

A strategic choice was made to build this bot using a **Custom Backend (Azure Container Apps)** rather than using low-code alternatives.

| Feature | Custom Setup (This Project) | Copilot Studio | Azure AI Foundry |
| :--- | :--- | :--- | :--- |
| **Logic Control** | **High**: Precise KQL generation & error handling. | **Medium**: Restricted to low-code canvas. | **N/A**: Dev platform, not hosting. |
| **Identity Flow** | **Zero-Secret OBO**: Full user impersonation trust. | **Rigid**: Difficult to implement deep OBO. | **Context Only**: Focus on RAG/Lifecycle. |
| **UX & Cards** | **Adaptive Cards**: Full JSON schema control. | **Standard**: Limited to studio templates. | **N/A**: Requires separate bot code. |
| **Cost** | **Pay-as-you-go**: Highly scalable/efficient. | **Licensing**: Per-user/message based. | **Tiered**: Based on model/compute. |

**The Verdict**: For governance tasks requiring high-precision data retrieval (Azure Resource Graph) and strict security boundaries, a custom Python-based backend provides the necessary flexibility that low-code platforms currently lack.

---

## 📖 Documentation Map

To understand the architecture and deployment, refer to the following documents:

*   [agent.md](agent.md): **The "Brain"** - Design specifications, intent detection logic, and architectural diagrams.
*   [implementation.md](implementation.md): **The "Blueprint"** - Resource lists, Federated Identity setup, and Terraform snippets.

---

## 🛡️ Core Pillars

1.  **Zero-Secret Architecture**: Utilizes **Federated Identity** (OIDC) and User-Assigned Managed Identities. No client secrets or passwords are stored in the repo.
2.  **Performance Intelligence**: Optimized with **GPT-4o** for rapid and accurate KQL translation.
3.  **US Regional Compliance**: Standardized on US-based hosting (e.g., `eastus2`) to meet data residency requirements.

---

## 🖼️ Architecture at a Glance

![Technical Architecture](docs/images/architecture.png)
