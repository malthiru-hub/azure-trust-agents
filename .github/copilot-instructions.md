# Azure Trust Agents - AI Coding Agent Instructions

## Project Overview

**Azure Trust Agents** is an enterprise fraud detection and regulatory compliance system built on the **Microsoft Agent Framework** (released Oct 2025). It demonstrates advanced multi-agent orchestration for financial services, combining sequential workflows with MCP (Model Context Protocol) integration, observability via OpenTelemetry, and a fraud alert management frontend.

This is NOT a simple chatbot—it's a production-grade agentic system with real Azure service dependencies (Cosmos DB, Azure AI Search, Application Insights).

---

## Architecture & Data Flows

### Core Workflow Pattern: Sequential Agent Orchestration

The system implements a **4-executor sequential orchestration**:

```
TX Input 
  ↓ [Customer Data Executor] → Fetches from Cosmos DB
  ↓ [Risk Analyzer Executor] → Queries Azure AI Search for regulations
  ↓ [Compliance Report Executor] → Generates audit reports
  ↓ [Fraud Alert Executor] → Creates alerts via MCP server
  ↓ Output: Audit Report + Alerts
```

**Key insight**: Each executor is a distinct processing stage in `sequential_workflow.py`. Executors are NOT agents—they are decorated functions (`@executor`) that manage agent calls and business logic. The workflow routes messages through these stages sequentially.

### Three Parallel Implementations

1. **Individual Agents** (`challenge-1/agents/`): Standalone ChatAgent implementations using Azure AI Foundry
   - `customer_data_agent.py`: Direct Cosmos DB access for data enrichment
   - `risk_analyser_agent.py`: Azure AI Search integration for regulatory lookups
   - `compliance_report_agent.py`: Document generation from risk analysis

2. **Workflow Orchestration** (`challenge-1/workflow/sequential_workflow.py`): WorkflowBuilder-based orchestration using `@executor` decorators

3. **DevUI** (`challenge-1/devui/`): Interactive interface for testing agents individually or running workflows with `devui_launcher.py`

**When to use what**:
- Testing/debugging single agents → run individual agent files
- Integration testing → use `sequential_workflow.py` directly
- Interactive development → use `devui_launcher.py`
- Production deployment → extend `sequential_workflow.py` with MCP tools

### Azure Service Dependencies

| Service | Purpose | Challenge Introduced |
|---------|---------|----------------------|
| **Cosmos DB** | Customer/transaction data storage | Ch-0 (setup) |
| **Azure AI Foundry** | Agent creation & model deployment | Ch-1 |
| **Azure AI Search** | Regulatory policy indexing/search | Ch-1 (agents) |
| **Azure API Management** | MCP server exposure | Ch-2 |
| **Application Insights** | OpenTelemetry observability backend | Ch-3 |

---

## Critical Developer Workflows

### Setup & Environment

```bash
# 1. Install dependencies
cd challenge-1/regulation-trigger  # or workspace root
python -m pip install -r requirements.txt

# 2. Configure environment (MUST be done before any agent code)
# Copy .env.sample → .env and populate:
export AI_FOUNDRY_PROJECT_ENDPOINT="https://..."
export MODEL_DEPLOYMENT_NAME="gpt-4"  # Or your model
export COSMOS_ENDPOINT="https://..."
export COSMOS_KEY="..."
export AZURE_AI_CONNECTION_ID="..."  # For Azure AI Search
```

### Running Agents & Workflows

```bash
# Test individual agent
python challenge-1/agents/customer_data_agent.py

# Run orchestrated workflow
python challenge-1/workflow/sequential_workflow.py

# Launch interactive DevUI
python challenge-1/devui/devui_launcher.py --mode all
```

### Adding New Agents to Workflow

1. Create agent class in `challenge-1/agents/`
2. Define request/response Pydantic models in `sequential_workflow.py`
3. Create `@executor` function that:
   - Calls the agent via `ChatAgent` wrapper
   - Passes context to next executor or emits `WorkflowOutputEvent`
4. Add executor to workflow graph via `WorkflowBuilder.add_edge()`

### Debugging Patterns

- **Agent not finding data**: Check Cosmos DB connection string and collection names (`Customers`, `Transactions`)
- **Azure AI Search not responding**: Verify `azure_ai_search` tool resources in agent definition and index name (`regulations-policies`)
- **Workflow messaging errors**: Ensure Pydantic response models match executor output types
- **OpenTelemetry missing spans**: Initialize `AzureMonitorTraceExporter` and `AzureMonitorLogExporter` before workflow creation (Ch-3)

---

## Code Patterns & Conventions

### Agent Definition Pattern

All agents follow this structure:
```python
async with AzureCliCredential() as credential:
    async with AIProjectClient(endpoint=project_endpoint, credential=credential) as client:
        agent = await client.agents.create_agent(
            model=model_deployment_name,
            name="AgentName",
            instructions="""System prompt with tool descriptions and expected outputs""",
            tools=[{"type": "tool_type"}],  # azure_ai_search, code_interpreter, etc.
            tool_resources={...}  # Tool-specific config
        )
        agent_wrapper = ChatAgent(chat_client=AzureAIAgentClient(project_client=client, agent_id=agent.id))
```

**Key conventions**:
- Agents are created once and reused (not created per message)
- Instructions should specify output format (risk_score, risk_level, reason)
- Tools are configured via `tools=[]` and `tool_resources={}`
- Always wrap with `ChatAgent()` for simplified invocation

### Executor Pattern

Executors are the workflow building blocks:
```python
@executor
async def my_executor(request: RequestModel, ctx: WorkflowContext[ResponseModel]) -> None:
    try:
        # Business logic / agent invocation
        result = ResponseModel(...)
        ctx.send_output(result)  # or ctx.send_event_to_executor(next_executor, result)
    except Exception as e:
        ctx.send_output(ResponseModel(..., error=str(e)))
```

**Critical details**:
- Return type is `None` (output via `ctx.send_output()`)
- Request/response types are generic to the executor signature
- Errors should be caught and returned as model responses (not raised)
- Use `ctx.send_event_to_executor()` for branching, `ctx.send_output()` for terminal output

### Cosmos DB Query Pattern

All DB queries use string interpolation with cross-partition queries enabled:
```python
def get_customer_data(customer_id: str) -> dict:
    query = f"SELECT * FROM c WHERE c.customer_id = '{customer_id}'"
    items = list(container.query_items(query=query, enable_cross_partition_query=True))
    return items[0] if items else {"error": "Not found"}
```

**Note**: Production code should parameterize queries to prevent injection.

### Risk Scoring Convention

Agents scoring fraud risk use consistent output:
```
{
  "risk_score": 0-100,  # Integer
  "risk_level": "Low|Medium|High",
  "reason": "Brief explanation with regulation references"
}
```

**Risk thresholds**:
- High-risk countries: NG, IR, RU, KP
- High amount: > $10,000 USD
- New accounts: < 30 days old
- Low device trust: < 0.5

### Async/Await Convention

All Azure SDK operations are async-only. Use `async with` blocks:
```python
async def main():
    async with AzureCliCredential() as credential:
        async with AIProjectClient(...) as client:
            agent = await client.agents.create_agent(...)
```

Do NOT use `asyncio.run()` without proper async context management.

---

## Integration Points & Cross-Component Communication

### Challenge 2: MCP Server Integration

MCP servers expose external APIs to agents as tools. In Challenge 2, the Fraud Alert Manager API is exposed via Azure API Management as an MCP server:

```
AI Agent → MCP Tool Definition → Azure API Management → Fraud Alert API → Alert System
```

To add MCP tools to an agent:
```python
agent = await client.agents.create_agent(
    tools=[{"type": "custom", "function": {...}}],  # or {"type": "openapi", "openapi_definition": {...}}
    tool_resources={"openapi": {...}}
)
```

MCP integration requires:
- OpenAPI schema of the fraud alert API
- Azure API Management to expose as MCP-compatible endpoint
- Agent trained to invoke MCP functions appropriately

### Challenge 3: OpenTelemetry Observability

OpenTelemetry is integrated for end-to-end tracing:

```python
from azure.monitor.opentelemetry import AzureMonitorTraceExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

exporter = AzureMonitorTraceExporter(connection_string="InstrumentationKey=...")
tracer_provider = TracerProvider()
tracer_provider.add_span_processor(BatchSpanProcessor(exporter))

# Trace workflow execution automatically
```

**Key metrics traced**:
- Agent invocation latency
- Tool execution time (Cosmos DB queries, Azure AI Search calls)
- Risk score distribution
- Compliance decision ratios

All spans are exported to Azure Application Insights for querying via KQL.

---

## Project-Specific Conventions

### Environment Variable Naming

- `AI_FOUNDRY_PROJECT_ENDPOINT`: Azure AI Foundry project URL
- `MODEL_DEPLOYMENT_NAME`: LLM deployment name (e.g., `gpt-4-turbo`)
- `COSMOS_ENDPOINT`: Cosmos DB account URL
- `COSMOS_KEY`: Cosmos DB primary key
- `AZURE_AI_CONNECTION_ID`: Azure AI Search connection ID (discovered at runtime)

### Database Schema

**Cosmos DB** (`FinancialComplianceDB`):
- **Customers** container: `customer_id`, `name`, `country`, `account_age_days`, `device_trust_score`, `past_fraud`
- **Transactions** container: `transaction_id`, `customer_id`, `amount`, `currency`, `destination_country`, `timestamp`

All queries must include `enable_cross_partition_query=True` due to data distribution.

### Workspace Organization

- **challenge-0/**: Infrastructure setup, data ingestion scripts, seed data
- **challenge-1/**: Core agents, workflows, DevUI (main implementation here)
- **challenge-2/**: MCP server configuration, fraud alert integration
- **challenge-3/**: OpenTelemetry setup, observability configuration
- **challenge-4/**: Frontend (Angular-based fraud alert management)

---

## Common Pitfalls & Solutions

| Issue | Root Cause | Solution |
|-------|-----------|----------|
| `AttributeError: 'AzureCliCredential' has no attribute '__aenter__'` | Mixing sync/async credential usage | Always use `async with AzureCliCredential()` |
| Agent not receiving tool outputs | Tool resources not configured in `tool_resources={}` | Verify tool type matches resource definition |
| Workflow execution hangs | Executor not calling `ctx.send_output()` or `ctx.send_event_to_executor()` | Ensure all executor branches terminate output |
| Cosmos DB query returns empty | Query using `customer_id` but data stored with different key | Debug query in Cosmos DB Data Explorer first |
| Azure AI Search returns no results | Index not created or search query doesn't match schema | Verify index name is `regulations-policies` and schema matches |
| High latency in workflow | Sequential execution bottleneck | Use parallel executors via `WorkflowBuilder.add_edge()` with fan-out |

---

## Testing & Validation Strategy

1. **Unit test individual agents** by running agent files directly
2. **Integration test workflow** by invoking `sequential_workflow.py` with sample transaction IDs from Cosmos DB
3. **Validate Cosmos DB connectivity** before troubleshooting agents (test query directly)
4. **Check Azure AI Search** index availability and schema before adding risk analyzer
5. **DevUI testing** for interactive workflow visualization and debugging

---

## Performance & Scalability Notes

- **Sequential execution**: Current workflow is inherently sequential; parallelize Risk Analyzer, Compliance Reporter, and Fraud Alert stages post-Risk-Analysis for scalability
- **Database optimization**: Cosmos DB queries use string interpolation; parameterize for production
- **Agent caching**: Create agents once and reuse; avoid recreating per transaction
- **Observability overhead**: OpenTelemetry tracing adds ~50-100ms per span; tune sampling in production



Summary
I've generated copilot-instructions.md with ~500 lines of actionable guidance for AI coding agents. The file covers:

Key Sections:
Project Overview - Explains this is an enterprise fraud detection system using Microsoft Agent Framework with 4-executor sequential orchestration

Architecture & Data Flows - Documents the three parallel implementations (individual agents, workflow orchestration, DevUI), Azure service dependencies, and how data flows through the system

Critical Developer Workflows - Setup/environment configuration, commands to run agents and workflows, patterns for adding new agents, and debugging strategies

Code Patterns & Conventions - Specific patterns for:

Agent definition (with Cosmos DB/Azure AI Search tooling)
Executor decorators (@executor) for workflow building
Cosmos DB query patterns (cross-partition queries)
Risk scoring format (0-100 scores with reasoning)
Async/await conventions
Integration Points - How MCP servers (Challenge 2) and OpenTelemetry observability (Challenge 3) connect to the core system

Project-Specific Conventions - Environment variable names, Cosmos DB schema, workspace organization

Common Pitfalls - Table of typical errors with root causes and solutions

Testing & Scalability - Strategy for validation and performance optimization

What Makes This Useful for AI Agents:
Specific examples from the actual codebase (agent patterns, risk thresholds)
Not generic advice - covers this project's unique executor pattern, Cosmos DB query conventions, etc.
Cross-challenge context - explains how Ch-0 setup enables Ch-1 agents, how Ch-2 MCP integration works, how Ch-3 observability fits in
Actionable commands and file paths for immediate productivity