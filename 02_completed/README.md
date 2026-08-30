# Travel Multi-Agent Workshop - Complete Solution

This directory contains the complete implementation of the Travel Multi-Agent Workshop: a fully functional multi-agent travel assistant built with Python, LangGraph, Azure AI Foundry, and Azure Cosmos DB.

Use this solution to:

- Deploy and run the application as a standalone demo.
- Explore a reference implementation while completing the workshop exercises.
- Demonstrate agent analytics and optimization with Microsoft Fabric.

## Getting Started

Deploy the complete solution 👉 **[Deploy to Azure](#deploy-to-azure)**

📖 **[User & Demo Guide](./USER_GUIDE.md)** - an end-to-end runbook for configuring, deploying, and demonstrating the application, agents, memory, analytics, and optimization features.

📊 **[Travel Multi-Agent Analytics Guide](https://github.com/AzureCosmosDB/cosmos-fabric-samples/tree/main/travel-multi-agent-analytics)** - generate realistic application data and analyze multi-agent memory, trip planning, and user preferences with Microsoft Fabric and Power BI.

## Deploy to Azure

To deploy the complete travel multi-agent assistant to your Azure account:

1. **Clone the `main` branch** and switch to this folder:
   ```bash
   git clone --branch main https://github.com/AzureCosmosDB/travel-multi-agent-workshop.git
   cd travel-multi-agent-workshop/02_completed
   ```
2. **Install prerequisites:**
   - [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)
   - [Azure Developer CLI (azd)](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd)
   - [Python 3.11+](https://www.python.org/downloads/)
   - [Node.js 18+ and npm](https://nodejs.org/en/download/)
3. **Log in to Azure:**
   ```bash
   azd auth login
   ```
4. **Provision and deploy:**
   ```bash
   azd up
   ```

   `azd up` provisions the Azure resources, seeds Cosmos DB, builds the application images, and deploys the frontend, API, and MCP server to Azure Container Apps. Provisioning takes several minutes. When deployment finishes, open the public `FRONTEND_URI` printed by `azd`.

See [Deployment and run options](#deployment-and-run-options) for optional flags and the local three-terminal flow.

## Deployment and run options

`azd provision` deploys the data and AI infrastructure: an Azure Cosmos DB account with the `TravelAssistant` database and an Azure AI Foundry account with the `gpt-5.1` chat model, `text-embedding-3-small`, and the optimization-tier models (`gpt-5-nano` and `gpt-5-mini`).

The post-provision hook writes `python/.env` and `mcp_server/.env`, creates the Python virtual environment, and seeds Cosmos DB. You can then run the application locally against the provisioned infrastructure instead of building and deploying containers on every change.

### Optional deployment flags

Set these Azure Developer CLI environment variables with `azd env set <NAME> <value>` before running `azd provision` or `azd up`:

| Flag (environment variable) | Default in `02_completed` | Effect |
|---|---:|---|
| `deployAnalytics` (`DEPLOY_ANALYTICS`) | **true** | Provisions the analytics and optimization Cosmos containers (`OptimizationPolicies`, `OptimizationTurns`, `OptimizationInsights`, and `Configuration`) and the Microsoft Fabric F2 capacity used by Modules 07-09. Set this to `false` for a leaner base deployment. |
| `deployGsi` (`DEPLOY_GSI`) | **false** | Provisions an alternative provisioned-throughput Cosmos DB account with a `TripsByDestination` global secondary index and seeds about 25,000 trips. This optional scaling demonstration is not required by any module. |
| `deployHostedApp` (`DEPLOY_HOSTED_APP`) | **true** | Deploys the frontend, API, and MCP server as hosted Azure Container Apps, together with Azure Container Registry, a Container Apps environment, and Log Analytics. Set this to `false` to skip hosting and run locally. |

This is the complete demo solution, so analytics and application hosting are enabled by default. Run `azd provision` instead of `azd up` if you only want the infrastructure and prefer the local flow.

```powershell
# Example: provision the base infrastructure without Fabric analytics.
azd env set DEPLOY_ANALYTICS false
azd provision
```

> **AI models and pricing:** `azd up` deploys `gpt-5.1`, `gpt-5-mini`, and `gpt-5-nano`, then seeds their token prices into the Cosmos DB `Configuration` container. The application, Fabric notebook, and Power BI report use these values to calculate turn costs consistently. If you change the deployed models, add each model's price to `python/data/model_pricing.json`. See [Model pricing](../analytics/docs/model-pricing.md) for the default models, price format, and instructions.

## Local development

From `02_completed/`, run `azd provision` first. The post-provision hook creates `.venv-travel` and configures the required environment files.

Open three terminals:

```powershell
# Terminal 1 - MCP server
.\.venv-travel\Scripts\Activate.ps1
cd mcp_server
$env:PYTHONPATH="..\python"
python mcp_http_server.py
```

```powershell
# Terminal 2 - Travel API
.\.venv-travel\Scripts\Activate.ps1
cd python
uvicorn src.app.travel_agents_api:app --reload --host 0.0.0.0 --port 8000
```

```powershell
# Terminal 3 - Angular frontend
cd frontend
npm install
npm start
```

Access the local applications:

- Travel API docs: [http://localhost:8000/docs](http://localhost:8000/docs)
- MCP server: [http://localhost:8080](http://localhost:8080)
- Frontend: [http://localhost:4200](http://localhost:4200)

## Configure, run, and demo this solution

See the **[User & Demo Guide](./USER_GUIDE.md)** for the complete demo workflow:

- Configure and deploy the solution.
- Stand up the Fabric analytics environment.
- Run the application and web Analytics Portal.
- Optionally connect the Power BI report.
- Generate traffic and walk through the optimization and translytical demo.

For a focused walkthrough of data generation and analysis, use the **[Travel Multi-Agent Analytics Guide](https://github.com/AzureCosmosDB/cosmos-fabric-samples/tree/main/travel-multi-agent-analytics)**.

## Memory layer

Memory is provided by the [`azure-cosmos-agent-memory`](https://pypi.org/project/azure-cosmos-agent-memory/) SDK. The toolkit creates the Cosmos DB `memories`, `memories_turns`, and `memories_summaries` containers on first run, so no Bicep container resources are required for memory.

Every 10 chat turns, a background auto-flush produces summaries, facts, and a `user_summary`. Memory records are partitioned by `(user_id, thread_id)`. `tenantId` remains part of sessions, messages, and trips but is not part of memory records. Memory prompts ship inside the toolkit rather than as `.prompty` files in this repository.

## Project structure

```text
02_completed/
├── python/       # Fully implemented Python application
│   ├── data/     # Sample data and seed scripts
│   └── src/      # Application source code
├── frontend/     # Angular web application
├── infra/        # Azure infrastructure as code
└── mcp_server/   # FastMCP tool server
```

This directory contains the complete implementation of the workshop modules, including multi-agent orchestration, persistent memory, Azure integrations, analytics, and optimization features.
