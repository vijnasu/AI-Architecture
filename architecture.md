# Enterprise AI Architecture — External LLMs, Local LLMs & RAG

End-to-end reference architecture combining external LLM providers, self-hosted/local LLMs, and a Retrieval-Augmented Generation (RAG) pipeline for enterprise use.

## Diagram

```mermaid
%%{init: {
  'theme': 'default',
  'themeVariables': {
    'primaryColor': '#EAF2FF',
    'primaryTextColor': '#102A43',
    'primaryBorderColor': '#2F6FED',
    'lineColor': '#486581',
    'secondaryColor': '#FFF3D6',
    'tertiaryColor': '#E8F5E9'
  },
  'flowchart': {
    'curve': 'basis',
    'htmlLabels': true
  },
  'securityLevel': 'loose',
  'zoom': true,
  'pan': { 'enabled': true, 'mode': 'drag' }
}}%%
flowchart TB
    subgraph Client["Client Layer"]
        U1["Azure App Service / Web App"]
        U2["Copilot / Internal Portal"]
        U3["B2B API Clients"]
    end

    subgraph Gateway["Azure API & Security"]
        GW["Azure API Management"]
        WAF["Azure Front Door / WAF"]
        IAM["Microsoft Entra ID"]
        DLP["Microsoft Purview / DLP"]
    end

    subgraph Orchestration["AI Orchestration"]
        ORCH["Azure Container Apps / AKS<br/>App Runtime"]
        ROUTER{"Model Router<br/>(policy, latency, cost, sensitivity)"}
        CACHE["Redis / Semantic Cache"]
        PROMPT["Prompt & Guardrails Engine"]
    end

    subgraph RAG["RAG Pipeline"]
        direction TB
        subgraph Ingestion["Data Ingestion"]
            LOAD["Azure AI Document Intelligence<br/>SharePoint / ADLS / SQL Connectors"]
            CHUNK["Chunking / Enrichment"]
            EMBED_IDX["Embedding Model<br/>(Azure OpenAI / Model Catalog)"]
        end
        VDB[("Azure AI Search<br/>Vector + Hybrid Search")]
        RETRIEVER["Retriever + Reranker"]
        EMBED_Q["Query Embedding Model"]
    end

    subgraph Models["Model Layer"]
        subgraph LocalLLM["Private / Local Models"]
            L1["Azure ML / AKS<br/>Self-hosted LLM"]
            L2["Fine-tuned Domain Model"]
            GPU["NVIDIA GPU Nodes"]
        end
        subgraph ExternalLLM["External / Managed Models"]
            E1["Azure OpenAI"]
            E2["Azure AI Foundry Models"]
        end
    end

    subgraph DataSources["Enterprise Data Sources"]
        DS1[("Azure SQL / Cosmos DB")]
        DS2[("SharePoint / OneDrive / ADLS")]
        DS3[("SAP / Salesforce / APIs")]
    end

    subgraph Governance["Observability & Governance"]
        LOG["Application Insights<br/>Log Analytics"]
        EVAL["Evaluation / Feedback Loop"]
        MON["Azure Cost Management"]
        AUDIT["Microsoft Purview / Defender"]
    end

    U1 --> GW
    U2 --> GW
    U3 --> GW
    GW --> WAF
    WAF --> IAM
    IAM --> DLP
    DLP --> ORCH

    ORCH --> PROMPT
    ORCH --> CACHE
    ORCH --> ROUTER
    ORCH --> RETRIEVER

    RETRIEVER --> EMBED_Q --> VDB
    VDB --> RETRIEVER
    LOAD --> CHUNK --> EMBED_IDX --> VDB
    DS1 --> LOAD
    DS2 --> LOAD
    DS3 --> LOAD
    RETRIEVER -- "grounded context" --> ORCH

    ROUTER -- "sensitive / offline / regulated" --> L1
    ROUTER -- "general-purpose / high-capability" --> E1
    L1 --- GPU
    L2 --- GPU
    E1 --> ORCH
    E2 --> ORCH
    L1 --> ORCH
    L2 --> ORCH
    ORCH --> GW
    GW --> U1
    GW --> U2
    GW --> U3

    ORCH -.-> LOG
    ROUTER -.-> MON
    ORCH -.-> EVAL
    GW -.-> AUDIT

    classDef client fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:#0f172a;
    classDef gateway fill:#fee2e2,stroke:#dc2626,stroke-width:2px,color:#7f1d1d;
    classDef orch fill:#e9d5ff,stroke:#7c3aed,stroke-width:2px,color:#2e1065;
    classDef ragProc fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d;
    classDef ragStore fill:#bbf7d0,stroke:#15803d,stroke-width:2px,color:#052e16;
    classDef local fill:#dbeafe,stroke:#0284c7,stroke-width:2px,color:#082f49;
    classDef external fill:#fef3c7,stroke:#d97706,stroke-width:2px,color:#451a03;
    classDef datasource fill:#e5e7eb,stroke:#475569,stroke-width:2px,color:#111827;
    classDef gov fill:#fef9c3,stroke:#ca8a04,stroke-width:2px,color:#422006;

    class U1,U2,U3 client;
    class GW,WAF,IAM,DLP gateway;
    class ORCH,ROUTER,CACHE,PROMPT orch;
    class LOAD,CHUNK,EMBED_IDX,RETRIEVER,EMBED_Q ragProc;
    class VDB ragStore;
    class L1,L2,GPU local;
    class E1,E2 external;
    class DS1,DS2,DS3 datasource;
    class LOG,EVAL,MON,AUDIT gov;
```

## Diagram Evolution

### 1) Original Generic Architecture

```mermaid
flowchart TB
    subgraph Client["Client Layer"]
        U1["Web / Mobile App"]
        U2["Internal Tools / Copilot Plugins"]
        U3["Third-Party API Consumers"]
    end

    subgraph Gateway["API Gateway & Security"]
        GW["API Gateway<br/>(Auth, Rate Limiting, WAF)"]
        IAM["IAM / SSO / OAuth2"]
        DLP["PII Redaction / DLP Filter"]
    end

    subgraph Orchestration["Orchestration Layer"]
        ORCH["AI Orchestrator<br/>(LangChain / Semantic Kernel / Custom)"]
        ROUTER{"Model Router<br/>(policy, cost, latency, sensitivity)"}
        CACHE["Semantic Cache"]
        PROMPT["Prompt Template & Guardrails Engine"]
    end

    subgraph RAG["RAG Pipeline"]
        direction TB
        subgraph Ingestion["Data Ingestion"]
            LOAD["Document Loaders<br/>(SharePoint, Confluence, DBs, Files)"]
            CHUNK["Chunking & Preprocessing"]
            EMBED_IDX["Embedding Model<br/>(Indexing)"]
        end
        VDB[("Vector Database<br/>(Azure AI Search / Pinecone / pgvector)")]
        RETRIEVER["Retriever<br/>(hybrid search + reranking)"]
        EMBED_Q["Embedding Model<br/>(Query)"]
    end

    subgraph Models["Model Layer"]
        subgraph LocalLLM["Local / Private LLMs"]
            L1["Self-Hosted LLM<br/>(Llama, Mistral, Phi via vLLM/TGI)"]
            L2["Fine-Tuned Domain Model"]
            GPU["GPU Cluster / On-Prem Inference"]
        end
        subgraph ExternalLLM["External LLM Providers"]
            E1["OpenAI / Azure OpenAI"]
            E2["Anthropic Claude"]
            E3["Google Gemini"]
        end
    end

    subgraph DataSources["Enterprise Data Sources"]
        DS1[("Structured DB<br/>(SQL/NoSQL)")]
        DS2[("Document Stores<br/>(SharePoint, Wiki)")]
        DS3[("APIs / SaaS Systems")]
    end

    subgraph Governance["Observability, Governance & Ops"]
        LOG["Logging & Tracing<br/>(OpenTelemetry)"]
        EVAL["Evaluation & Feedback Loop"]
        MON["Cost & Usage Monitoring"]
        AUDIT["Compliance / Audit Trail"]
    end

    U1 --> GW
    U2 --> GW
    U3 --> GW
    GW --> IAM
    GW --> DLP
    DLP --> ORCH
    ORCH --> PROMPT
    ORCH --> CACHE
    ORCH --> ROUTER
    ORCH --> RETRIEVER
    RETRIEVER --> EMBED_Q --> VDB
    VDB --> RETRIEVER
    LOAD --> CHUNK --> EMBED_IDX --> VDB
    DS1 --> LOAD
    DS2 --> LOAD
    DS3 --> LOAD
    RETRIEVER -- "augmented context" --> ORCH
    ROUTER -- "sensitive / offline data" --> L1
    ROUTER -- "general / high-capability tasks" --> E1
    L1 --- GPU
    L2 --- GPU
    E1 --> ORCH
    E2 --> ORCH
    E3 --> ORCH
    L1 --> ORCH
    L2 --> ORCH
    ORCH --> GW
    GW --> U1
    GW --> U2
    GW --> U3
    ORCH -.-> LOG
    ROUTER -.-> MON
    ORCH -.-> EVAL
    GW -.-> AUDIT
```

### 2) Improved Azure Architecture

```mermaid
%%{init: {'theme': 'default', 'themeVariables': { 'primaryColor': '#EAF2FF', 'primaryTextColor': '#102A43', 'primaryBorderColor': '#2F6FED', 'lineColor': '#486581', 'secondaryColor': '#FFF3D6', 'tertiaryColor': '#E8F5E9'}}}%%
flowchart TB
    subgraph Client["Client Layer"]
        U1["Azure App Service / Web App"]
        U2["Copilot / Internal Portal"]
        U3["B2B API Clients"]
    end

    subgraph Gateway["Azure API & Security"]
        GW["Azure API Management"]
        WAF["Azure Front Door / WAF"]
        IAM["Microsoft Entra ID"]
        DLP["Microsoft Purview / DLP"]
    end

    subgraph Orchestration["AI Orchestration"]
        ORCH["Azure Container Apps / AKS<br/>App Runtime"]
        ROUTER{"Model Router<br/>(policy, latency, cost, sensitivity)"}
        CACHE["Redis / Semantic Cache"]
        PROMPT["Prompt & Guardrails Engine"]
    end

    subgraph RAG["RAG Pipeline"]
        direction TB
        subgraph Ingestion["Data Ingestion"]
            LOAD["Azure AI Document Intelligence<br/>SharePoint / ADLS / SQL Connectors"]
            CHUNK["Chunking / Enrichment"]
            EMBED_IDX["Embedding Model<br/>(Azure OpenAI / Model Catalog)"]
        end
        VDB[("Azure AI Search<br/>Vector + Hybrid Search")]
        RETRIEVER["Retriever + Reranker"]
        EMBED_Q["Query Embedding Model"]
    end

    subgraph Models["Model Layer"]
        subgraph LocalLLM["Private / Local Models"]
            L1["Azure ML / AKS<br/>Self-hosted LLM"]
            L2["Fine-tuned Domain Model"]
            GPU["NVIDIA GPU Nodes"]
        end
        subgraph ExternalLLM["External / Managed Models"]
            E1["Azure OpenAI"]
            E2["Azure AI Foundry Models"]
        end
    end

    subgraph DataSources["Enterprise Data Sources"]
        DS1[("Azure SQL / Cosmos DB")]
        DS2[("SharePoint / OneDrive / ADLS")]
        DS3[("SAP / Salesforce / APIs")]
    end

    subgraph Governance["Observability & Governance"]
        LOG["Application Insights<br/>Log Analytics"]
        EVAL["Evaluation / Feedback Loop"]
        MON["Azure Cost Management"]
        AUDIT["Microsoft Purview / Defender"]
    end

    U1 --> GW
    U2 --> GW
    U3 --> GW
    GW --> WAF
    WAF --> IAM
    IAM --> DLP
    DLP --> ORCH
    ORCH --> PROMPT
    ORCH --> CACHE
    ORCH --> ROUTER
    ORCH --> RETRIEVER
    RETRIEVER --> EMBED_Q --> VDB
    VDB --> RETRIEVER
    LOAD --> CHUNK --> EMBED_IDX --> VDB
    DS1 --> LOAD
    DS2 --> LOAD
    DS3 --> LOAD
    RETRIEVER -- "grounded context" --> ORCH
    ROUTER -- "sensitive / offline / regulated" --> L1
    ROUTER -- "general-purpose / high-capability" --> E1
    L1 --- GPU
    L2 --- GPU
    E1 --> ORCH
    E2 --> ORCH
    L1 --> ORCH
    L2 --> ORCH
    ORCH --> GW
    GW --> U1
    GW --> U2
    GW --> U3
    ORCH -.-> LOG
    ROUTER -.-> MON
    ORCH -.-> EVAL
    GW -.-> AUDIT
```

### 3) Final Polished Interactive Version

This is the current version in the main diagram above, with improved readability, color coding, and mouse-friendly pan/zoom enabled.

## Layer Descriptions

### Client Layer
Entry points for end users and systems: web/mobile apps, internal tooling (e.g. Copilot plugins), and third-party API consumers.

### API Gateway & Security
- **API Gateway** — authentication, rate limiting, WAF protection.
- **IAM / SSO / OAuth2** — identity and access control.
- **PII Redaction / DLP Filter** — strips or masks sensitive data before it reaches the orchestrator.

### Orchestration Layer
- **AI Orchestrator** — coordinates prompt construction, retrieval, model routing, and response assembly (e.g. LangChain, Semantic Kernel).
- **Model Router** — chooses between local and external LLMs based on policy, cost, latency, and data sensitivity.
- **Semantic Cache** — avoids redundant model calls for repeated/similar queries.
- **Prompt Template & Guardrails Engine** — enforces prompt structure, safety filters, and output validation.

### RAG Pipeline
- **Ingestion** — document loaders pull from enterprise sources, chunk content, and generate embeddings for indexing.
- **Vector Database** — stores embeddings (e.g. Azure AI Search, Pinecone, pgvector).
- **Retriever** — hybrid search (vector + keyword) with reranking, triggered at query time via a query embedding model.

### Model Layer
- **Local / Private LLMs** — self-hosted or fine-tuned models (Llama, Mistral, Phi) served via vLLM/TGI on a GPU cluster; used for sensitive or offline workloads.
- **External LLM Providers** — OpenAI/Azure OpenAI, Anthropic Claude, Google Gemini; used for general-purpose, high-capability tasks.

### Enterprise Data Sources
Structured databases, document stores (SharePoint, wikis), and internal/external APIs feeding the ingestion pipeline.

### Observability, Governance & Ops
- **Logging & Tracing** — end-to-end request tracing (OpenTelemetry).
- **Evaluation & Feedback Loop** — quality scoring and human feedback capture.
- **Cost & Usage Monitoring** — tracks spend and usage per model/route.
- **Compliance / Audit Trail** — records access and decisions for regulatory needs.

## Routing Logic Summary
| Condition | Routed To |
|---|---|
| Sensitive/regulated data, offline requirement, cost-sensitive high-volume | Local LLM |
| General reasoning, high capability, low data sensitivity | External LLM |
