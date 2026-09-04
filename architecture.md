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

## Additional Cloud-Native Variants

### 4) AWS-Native Architecture

```mermaid
%%{init: {'theme': 'default', 'themeVariables': { 'primaryColor': '#F3F9FF', 'primaryTextColor': '#102A43', 'primaryBorderColor': '#2F6FED', 'lineColor': '#486581', 'secondaryColor': '#FFF3D6', 'tertiaryColor': '#E8F5E9'}}}%%
flowchart TB
    subgraph Client["Clients"]
        U1["Web / Mobile Apps"]
        U2["Internal Portal"]
        U3["B2B API Clients"]
    end

    subgraph Security["AWS Security & Access"]
        APIGW["API Gateway"]
        WAF["AWS WAF / CloudFront"]
        COG["Amazon Cognito"]
        IAM["IAM / AWS Organizations"]
        DLP["Macie / GuardDuty / Config"]
    end

    subgraph Runtime["AI Application Runtime"]
        ORCH["EKS / ECS / Fargate"]
        ROUTER{"Model Router<br/>(cost, latency, policy)"}
        CACHE["ElastiCache / Redis"]
        PROMPT["Prompt & Guardrail Service"]
    end

    subgraph RAG["RAG Layer"]
        DS1["S3 / Document Storage"]
        CHUNK["Document Parsing & Chunking"]
        EMBED["Embedding Service"]
        VDB[("OpenSearch Serverless / Bedrock Knowledge Base")]
        RETRIEVER["Retriever + Reranker"]
    end

    subgraph Models["Model Layer"]
        EXT["Amazon Bedrock<br/>Claude / Titan / Nova"]
        LOCAL["SageMaker / EKS<br/>Self-hosted LLM"]
        GPU["GPU-backed Training / Inference"]
    end

    subgraph Data["Enterprise Data Sources"]
        SQL[("RDS / Aurora")]
        DOCS[("S3 / SharePoint / Salesforce")]
        SAP[("SAP / ERP APIs")]
    end

    subgraph Governance["Observability & Governance"]
        LOG["CloudWatch / X-Ray"]
        MON["Cost Explorer"]
        AUDIT["AWS Audit Manager / Security Hub"]
    end

    U1 --> APIGW
    U2 --> APIGW
    U3 --> APIGW
    APIGW --> WAF
    WAF --> COG
    COG --> IAM
    IAM --> DLP
    DLP --> ORCH

    ORCH --> PROMPT
    ORCH --> CACHE
    ORCH --> ROUTER
    ORCH --> RETRIEVER

    DS1 --> CHUNK --> EMBED --> VDB
    SQL --> CHUNK
    DOCS --> CHUNK
    SAP --> CHUNK
    VDB --> RETRIEVER --> ORCH

    ROUTER --> LOCAL
    ROUTER --> EXT
    LOCAL --> GPU
    EXT --> ORCH
    LOCAL --> ORCH

    ORCH --> APIGW
    APIGW --> U1
    APIGW --> U2
    APIGW --> U3

    ORCH -.-> LOG
    ROUTER -.-> MON
    ORCH -.-> AUDIT
```

### 5) GCP-Native Architecture

```mermaid
%%{init: {'theme': 'default', 'themeVariables': { 'primaryColor': '#F0FDF4', 'primaryTextColor': '#102A43', 'primaryBorderColor': '#16A34A', 'lineColor': '#486581', 'secondaryColor': '#EFF6FF', 'tertiaryColor': '#ECFCCB'}}}%%
flowchart TB
    subgraph Client["Clients"]
        U1["Web / Mobile Apps"]
        U2["Internal Portal"]
        U3["B2B API Clients"]
    end

    subgraph Security["Google Cloud Security"]
        LB["Cloud Load Balancer"]
        GATEWAY["API Gateway / IAP"]
        IAM["IAM / Identity Platform"]
        DLP["Cloud DLP / Security Command Center"]
    end

    subgraph Runtime["AI Application Runtime"]
        ORCH["Cloud Run / GKE"]
        ROUTER{"Model Router<br/>(latency, policy, cost)"}
        CACHE["Redis / Memorystore"]
        PROMPT["Prompt & Policy Layer"]
    end

    subgraph RAG["RAG Layer"]
        DOCS["Cloud Storage / BigQuery"]
        CHUNK["Document Processing"]
        EMBED["Embedding Service"]
        VDB[("Vertex AI Search / Vector Index")]
        RETRIEVER["Retriever + Reranker"]
    end

    subgraph Models["Model Layer"]
        EXT["Vertex AI / Gemini"]
        LOCAL["GKE + GPUs<br/>Self-hosted LLM"]
    end

    subgraph Data["Enterprise Data Sources"]
        SQL[("Cloud SQL / Spanner")]
        DRIVE[("Drive / SharePoint / ERP")]
    end

    subgraph Governance["Monitoring & Governance"]
        LOG["Cloud Logging / Monitoring"]
        EVAL["Vertex AI Eval"]
        AUDIT["Security Command Center"]
    end

    U1 --> LB
    U2 --> LB
    U3 --> LB
    LB --> GATEWAY
    GATEWAY --> IAM
    IAM --> DLP
    DLP --> ORCH

    ORCH --> PROMPT
    ORCH --> CACHE
    ORCH --> ROUTER
    ORCH --> RETRIEVER

    DOCS --> CHUNK --> EMBED --> VDB
    SQL --> CHUNK
    DRIVE --> CHUNK
    VDB --> RETRIEVER --> ORCH

    ROUTER --> EXT
    ROUTER --> LOCAL
    EXT --> ORCH
    LOCAL --> ORCH

    ORCH --> LB
    LB --> U1
    LB --> U2
    LB --> U3

    ORCH -.-> LOG
    ORCH -.-> EVAL
    ORCH -.-> AUDIT
```

### 6) Executive-Friendly “CIO View”

```mermaid
%%{init: {'theme': 'default', 'themeVariables': { 'primaryColor': '#F8FAFC', 'primaryTextColor': '#0F172A', 'primaryBorderColor': '#334155', 'lineColor': '#64748B', 'secondaryColor': '#E2E8F0', 'tertiaryColor': '#E0F2FE'}}}%%
flowchart LR
    B1["Business Apps"] --> A1["AI Platform"]
    A1 --> D1["Enterprise Knowledge"]
    A1 --> M1["LLM Model Layer"]
    M1 --> E1["External LLMs"]
    M1 --> L1["Private / Local LLMs"]
    D1 --> R1["RAG / Retrieval"]
    A1 --> G1["Governance & Security"]
    G1 --> C1["Compliance / Audit"]
    G1 --> O1["Observability / Cost"]
    A1 --> B2["Business Outcomes"]
    D1 --> B2

    classDef business fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:#0f172a;
    classDef platform fill:#e9d5ff,stroke:#7c3aed,stroke-width:2px,color:#2e1065;
    classDef data fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d;
    classDef model fill:#fef3c7,stroke:#d97706,stroke-width:2px,color:#451a03;
    classDef gov fill:#fef9c3,stroke:#ca8a04,stroke-width:2px,color:#422006;

    class B1,B2 business;
    class A1,R1 platform;
    class D1 data;
    class M1,E1,L1 model;
    class G1,C1,O1 gov;
```

### 7) Deeper Deployment Topology with VNet, Private Endpoints, and Network Zones

```mermaid
%%{init: {'theme': 'default', 'themeVariables': { 'primaryColor': '#EFF6FF', 'primaryTextColor': '#0F172A', 'primaryBorderColor': '#2563EB', 'lineColor': '#475569', 'secondaryColor': '#F0FDF4', 'tertiaryColor': '#FEF3C7'}}}%%
flowchart TB
    subgraph Internet["Public Internet"]
        U1["Users / Apps"]
    end

    subgraph Edge["Edge & Access Layer"]
        LB["Azure Front Door / WAF"]
        APIM["API Management"]
        ID["Entra ID / IAM"]
    end

    subgraph DMZ["DMZ / Network Boundary"]
        GW["Ingress Gateway"]
        NAT["NAT / Firewall"]
    end

    subgraph VNet["Virtual Network (Private)]
        subgraph AppSubnet["Application Tier"]
            APP["AI App / Agent Runtime"]
            ORCH["Model Router / Service Mesh"]
        end

        subgraph AISubnet["AI / Compute Tier"]
            LLM["Private LLM Runtime"]
            GPU["GPU Cluster"]
        end

        subgraph DataSubnet["Data & Retrieval Tier"]
            AISEARCH["AI Search / Vector DB"]
            SQL["Azure SQL / Cosmos DB"]
            KV["Key Vault"]
        end

        subgraph GovSubnet["Observability & Security"]
            LOG["Log Analytics / App Insights"]
            PURVIEW["Purview / Defender"]
        end
    end

    subgraph PE["Private Endpoints / Private Link"]
        PE1["Private Endpoint: SQL"]
        PE2["Private Endpoint: Storage"]
        PE3["Private Endpoint: AI Search"]
        PE4["Private Endpoint: Key Vault"]
    end

    subgraph Zones["Availability Zones"]
        Z1["Zone 1"]
        Z2["Zone 2"]
        Z3["Zone 3"]
    end

    U1 --> LB --> APIM --> ID
    ID --> GW --> NAT --> APP
    APP --> ORCH
    ORCH --> LLM
    LLM --> GPU
    ORCH --> AISEARCH
    ORCH --> SQL
    APP --> KV

    SQL --> PE1
    AISEARCH --> PE3
    KV --> PE4
    PE2 --> APP

    APP --> LOG
    ORCH --> PURVIEW

    Z1 --> APP
    Z2 --> ORCH
    Z3 --> LLM
```

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
