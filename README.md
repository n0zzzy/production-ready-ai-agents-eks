# Enterprise Production-Ready AI Agents on Amazon EKS & AWS Bedrock

An end-to-end, multi-tier enterprise platform demonstrating how to build, deploy, scale, and evaluate AI agents on **Amazon EKS**.

This project implements two architectural paradigms—**Track 1: Self-Managed (Open-Source on EKS)** and **Track 2: Integrated (Managed AWS Services)**. By establishing a decoupled **Shared Model Plane** via [LiteLLM](https://github.com/BerriAI/litellm), orchestration, tool calling, and business logic remain completely agnostic to the underlying foundation model backends.

---

## 🏗 High-Level Architecture

                              ┌────────────────────────┐
                              │   Chainlit UI Client   │
                              └───────────┬────────────┘
                                          │
                                          ▼
                             ┌──────────────────────────┐
                             │ Orchestrator Agent (A2A) │
                             └─────────────┬────────────┘
                                           │
                  ┌────────────────────────┴────────────────────────┐
                  ▼                                                 ▼
      ┌───────────────────────┐                         ┌───────────────────────┐
      │  Order Agent (MCP)    │                         │ Product Agent (Milvus)│
      └───────────┬───────────┘                         └───────────┬───────────┘
                  │                                                 │
                  └────────────────────────┬────────────────────────┘
                                           │
                                           ▼
                              ┌────────────────────────┐
                              │   LiteLLM AI Gateway   │
                              │ (EKS Pod Identity/IAM) │
                              └───────────┬────────────┘
                                          │
                ┌─────────────────────────┴─────────────────────────┐
                ▼                                                   ▼
   [Track 1: Self-Managed]                             [Track 2: Integrated]
┌────────────────────────────────┐                 ┌────────────────────────────────┐
│ vLLM on AWS Inferentia         │                 │ Amazon Bedrock                 │
│ - Qwen 2.5 3B (Neuron Core)    │                 │ - Amazon Nova Lite             │
│ - OpenTelemetry -> Langfuse    │                 │ - Claude Sonnet (LLM Judge)    │
│ - Vector RAG via Milvus        │                 │ - AgentCore Memory & Sandbox   │
└────────────────────────────────┘                 └────────────────────────────────┘


---

## 🔑 Core Architectural Tenet: The Shared Model Plane

A central design pattern in this architecture is the **Shared Model Plane**:
* **Universal OpenAI Protocol:** An in-cluster [LiteLLM Proxy](https://github.com/BerriAI/litellm) exposes an OpenAI-compatible wire format (`/v1/chat/completions`) to all agents.
* **Seamless Backend Swaps:** Switching from self-hosted vLLM on AWS Inferentia to managed Amazon Bedrock requires only a single line configuration change (`model_id="nova-lite"`), keeping agent code, MCP tool bindings, and prompt logic untouched.
* **Credential-Free Pods:** Authentication to Amazon Bedrock is offloaded to the proxy via **EKS Pod Identity**, eliminating static AWS credentials inside the application containers.

---

## 📊 Dual-Track Implementation Matrix

| Capability | Track 1: Self-Managed (OSS on EKS) | Track 2: Integrated (AWS Managed) |
| :--- | :--- | :--- |
| **Model Serving** | **Qwen 2.5 3B** served via **vLLM** on AWS Inferentia chips | **Amazon Nova Lite** via **Amazon Bedrock** |
| **Model Gateway** | **LiteLLM Proxy** on EKS | **LiteLLM Proxy** on EKS (Zero-code model swap) |
| **Agent Framework** | **Strands Agents SDK** (`OpenAIModel` provider) | **Strands Agents SDK** configured for Bedrock |
| **Semantic Search (RAG)** | **Milvus Standalone** + **fastembed** (`all-MiniLM-L6-v2` ONNX) | Bedrock Knowledge Bases / Hybrid RAG |
| **Tool Execution** | **Model Context Protocol (MCP)** Server on EKS | **AgentCore Browser** + **AgentCore Code Interpreter** |
| **Session Memory** | Conversational vector storage via **Milvus** | Serverless state via **AgentCore Memory** |
| **Multi-Agent** | Distributed **A2A** (Orchestrator + Specialists) | Cloud-managed **A2A** orchestration |
| **Observability** | **OpenTelemetry (OTel)** spans streamed into **Langfuse** | Unified OTel spans into **Langfuse** |
| **Automated Evaluation** | **Claude Sonnet 4.5** on Bedrock (LLM-as-a-Judge) | **AgentCore Evaluation** native suite |

---

## 📂 Repository Structure

```text
├── modules/
│   ├── 20-self-managed/
│   │   ├── 200-strands-agents/          # Baseline Strands Agent with OpenAIModel format
│   │   ├── 300-observability-langfuse/   # OpenTelemetry collector & Langfuse exporter
│   │   ├── 400-rag-milvus/               # Vector catalog ingestion & semantic search
│   │   ├── 500-memory-milvus/            # Conversational state & session retrieval
│   │   ├── 600-agent-tools-mcp/          # Microservice MCP Server & tool endpoints
│   │   ├── 700-multi-agent-a2a/          # A2A Orchestrator, Order & Product agents
│   │   ├── 750-evaluation-llm-judge/     # Claude Sonnet 4.5 evaluation rubrics
│   │   └── 800-knowledge-graph/          # Graph RAG leveraging Neo4j
│   └── 30-integrated/
│       ├── 100-strands-bedrock/          # Strands integration targeting Amazon Bedrock
│       ├── 200-agentcore-memory/         # Stateful session memory with AgentCore
│       ├── 300-managed-capabilities/     # Sandboxed Code Interpreter & Browser
│       ├── 400-multi-agent-a2a/          # Cloud-managed Multi-Agent coordination
│       └── 500-agentcore-evaluation/     # Bedrock AgentCore evaluation suites
├── terraform/                            # EKS cluster, IAM roles, and Pod Identity
└── ui/                                   # Chainlit chat frontend for agent testing
