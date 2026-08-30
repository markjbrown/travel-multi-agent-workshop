# Travel Multi-Agent Workshop

## Overview

This workshop walks through how to build a multi-agent travel assistant using Python, LangGraph, Azure AI Foundry, and Azure Cosmos DB. You will create specialized AI agents that work together to help users plan travel and learn about agent memory orchestration.

The workshop concludes with optional modules on agent analytics and optimization: instrumenting agents, surfacing cost and quality insights, and applying and automatically reverting reversible optimizations. It also includes a Microsoft Fabric analytics and reverse ETL module.

![Cosmos Voyager - the multi-agent travel assistant web app](media/app-frontend.png)

*The Cosmos Voyager web app - explore places, plan trips, and chat with the multi-agent assistant.*

### What you will build

By the end of this workshop, you will have created a complete travel planning application featuring:

- **Multiple specialized agents:** Hotel booking, dining recommendation, activity planning, and other agents.
- **Intelligent orchestration:** A coordinator agent that manages interactions between specialized agents.
- **Memory system:** Persistent memory in Azure Cosmos DB that remembers user preferences and past interactions.
- **Modern web interface:** An Angular frontend that provides an intuitive chat experience.
- **API layer:** A FastAPI backend that orchestrates agent interactions.

### Memory layer

Memory is provided by the [`azure-cosmos-agent-memory`](https://pypi.org/project/azure-cosmos-agent-memory/) SDK. The toolkit creates its Cosmos DB `memories`, `memories_turns`, `memories_summaries`, and `counter` containers on first run through `connect_cosmos()`.

Auto-summarization thresholds are controlled by the `FACT_EXTRACTION_EVERY_N`, `DEDUP_EVERY_N`, `THREAD_SUMMARY_EVERY_N`, and `USER_SUMMARY_EVERY_N` environment variables. Memory records are partitioned by `(user_id, thread_id)`. Memory prompts ship inside the package rather than as `.prompty` files in this repository.

### Learning objectives

- Understand multi-agent architecture patterns and design principles.
- Build agents with LangGraph and Azure AI Foundry.
- Implement agent specialization and tool integration.
- Add intelligent memory systems to improve agent interactions.
- Practice observability and experimentation techniques.
- Deploy and manage AI applications on Azure.

## Optional: Agent Analytics and Optimization

Modules **07-09** are an optional track that turns the assistant into a self-observing system. You instrument every turn, then **detect, recommend, apply, and verify** cost and quality optimizations.

You can read and act on the results in a single-page web Analytics Portal in `analytics/dashboard/`. It includes live KPIs, per-agent scorecards, a conversion funnel, and one-click application or reversal of reversible policies. The portal is backed by Microsoft Fabric reverse ETL in Module 09 and includes an optional Power BI report.

![Analytics Portal - Overview tab](analytics/media/portal/portal-01-overview.png)

*The Analytics Portal Overview tab - portfolio KPIs, model usage, and live turn volume across the six optimization pillars.*

Not planning to complete the analytics track? Build the assistant in Modules 00-06, then use the exit ramp in Module 06 to jump to Module 10. You can return to Modules 07-09 later.

For a focused data-generation and analysis walkthrough, see the **[Travel Multi-Agent Analytics Guide](https://github.com/AzureCosmosDB/cosmos-fabric-samples/tree/main/travel-multi-agent-analytics)**.

## Getting started

This repository contains two main directories:

### 📚 `01_exercises` - Build the workshop solution

Follow the step-by-step workshop modules to build the application from scratch and learn each concept progressively.

Get started 👉 **[Start the Workshop](01_exercises/workshop/Home.md)**

### ✅ `02_completed` - Run the demo and explore the reference implementation

Use the fully implemented solution when you want to:

- Deploy a complete hosted demo to Azure.
- Run the application locally.
- Review the finished implementation while completing exercises.
- Demonstrate agent memory, analytics, optimization, Microsoft Fabric, and Power BI integration.

The deployment defaults are configured for the demo experience: `azd up` provisions and seeds the required services, then deploys the frontend, API, and MCP server to Azure Container Apps.

Run or explore the completed solution 👉 **[Open the Complete Solution](02_completed/README.md)**
