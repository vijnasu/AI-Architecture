# Enterprise AI Architecture: External LLMs, Local LLMs, and RAG

End-to-end reference architecture combining external managed LLMs, private or self-hosted LLMs, and a Retrieval-Augmented Generation (RAG) pipeline for enterprise use.

## Contents

- [Reference Architecture](#reference-architecture)
- [Architecture Evolution](#architecture-evolution)
- [Standardized AI SDLC](#standardized-ai-sdlc)
- [Cloud-Native Variants](#cloud-native-variants)
- [Executive CIO View](#executive-cio-view)
- [Deployment Topology](#deployment-topology)
- [Cloud Platform Comparison](#cloud-platform-comparison)
- [Architecture Layers](#architecture-layers)
- [Routing Logic](#routing-logic)

## Reference Architecture

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

## Architecture Evolution

### Original Generic Architecture

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

### Improved Azure Architecture

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

## Standardized AI SDLC

A standardized AI SDLC is the repeatable lifecycle used to design, build, validate, deploy, and govern enterprise AI systems with traceability and controls.

```mermaid
flowchart LR
    A["1. Business Goal & Risk"] --> B["2. Data & Governance"]
    B --> C["3. Model Selection"]
    C --> D["4. RAG / Prompting"]
    D --> E["5. Evaluation & Testing"]
    E --> F["6. Secure Deployment"]
    F --> G["7. Monitoring & Feedback"]
    G --> A

    classDef stage fill:#e0f2fe,stroke:#0284c7,stroke-width:2px,color:#082f49;
    class A,B,C,D,E,F,G stage;
```

This lifecycle governs the platform continuously rather than acting as a one-time project checklist:

- Business goals and risk classification precede model routing and deployment.
- Data governance applies before, during, and after RAG ingestion.
- Model selection is handled by the router across private and managed LLMs.
- Prompting, grounding, and retrieval are controlled by the guardrails layer.
- Evaluation uses automated scoring, human feedback, and safety checks before release.
- Secure deployment uses the gateway, identity, WAF, DLP, and audit controls.
- Monitoring closes the loop through telemetry, cost tracking, drift detection, and content refresh.

### SDLC Overlay on the Azure Architecture

```mermaid
flowchart LR
    subgraph SDLC["Standardized AI SDLC"]
        A["1. Business Goal & Risk"] --> B["2. Data & Governance"]
        B --> C["3. Model Selection"]
        C --> D["4. RAG / Prompting"]
        D --> E["5. Evaluation & Testing"]
        E --> F["6. Secure Deployment"]
        F --> G["7. Monitor & Feedback"]
        G --> A
    end

    subgraph ARCH["Azure AI Platform"]
        P1["Business Use Cases"]
        P2["Enterprise Data & Policies"]
        P3["Model Router"]
        P4["RAG + Guardrails"]
        P5["External / Private LLMs"]
        P6["API Gateway + WAF + Entra ID"]
        P7["Observability + Audit"]
    end

    A --> P1
    B --> P2
    C --> P3
    D --> P4
    D --> P5
    E --> P7
    F --> P6
    G --> P7
    P7 --> B

    classDef stage fill:#e0f2fe,stroke:#0284c7,stroke-width:2px,color:#082f49;
    classDef core fill:#e9d5ff,stroke:#7c3aed,stroke-width:2px,color:#2e1065;
    classDef data fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d;
    classDef sec fill:#fee2e2,stroke:#dc2626,stroke-width:2px,color:#7f1d1d;
    classDef obs fill:#fef9c3,stroke:#ca8a04,stroke-width:2px,color:#422006;

    class A,B,C,D,E,F,G stage;
    class P1,P2,P3,P4,P5 core;
    class P2 data;
    class P6 sec;
    class P7 obs;
```

This overlay shows the SDLC as the operating model that governs the platform. The architecture provides the execution path; the SDLC governs data, model choice, evaluation, deployment, and continuous improvement.

## Cloud-Native Variants

### AWS-Native Architecture

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

### GCP-Native Architecture

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

## Executive CIO View

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

## Deployment Topology

The following topology adds private networking, subnet boundaries, private endpoints, and availability zones for a regulated Azure deployment.

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

    subgraph VNet["Virtual Network (Private)"]
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

## Cloud Platform Comparison

| Capability | Azure | AWS | GCP |
|---|---|---|---|
| Identity & access | Microsoft Entra ID | Cognito + IAM | IAM + Identity Platform |
| Edge / ingress | Azure Front Door, API Management | CloudFront, API Gateway | Cloud Load Balancer, API Gateway / IAP |
| App runtime | Container Apps, AKS | EKS, ECS, Fargate | Cloud Run, GKE |
| Managed model layer | Azure OpenAI, Azure AI Foundry | Bedrock | Vertex AI, Gemini |
| Private / local models | Azure ML + AKS + GPU | SageMaker + EKS + GPU | GKE + GPU |
| Vector / RAG store | Azure AI Search | OpenSearch Serverless, Bedrock KB | Vertex AI Search, Vector Index |
| Structured data | Azure SQL, Cosmos DB | RDS, Aurora | Cloud SQL, Spanner |
| Object / document storage | ADLS, SharePoint | S3 | Cloud Storage |
| Governance | Purview, Defender | GuardDuty, Security Hub | Security Command Center |
| Observability | App Insights, Log Analytics | CloudWatch, X-Ray | Cloud Logging, Monitoring |
| Network isolation | VNet + Private Endpoints | VPC + PrivateLink | VPC + Private Service Connect |

## Architecture Layers

### Client Layer
Entry points for end users and systems: web applications, internal portals, enterprise copilots, and third-party API consumers.

### API Gateway & Security
- **API gateway** — authentication, rate limiting, and traffic control.
- **IAM / SSO / OAuth2** — identity and access control.
- **DLP and PII controls** — prevent sensitive data from reaching the orchestrator or model layer without policy enforcement.

### Orchestration Layer
- **AI orchestrator** — coordinates prompt construction, retrieval, model routing, and response assembly.
- **Model router** — chooses between local and external LLMs based on policy, cost, latency, and data sensitivity.
- **Semantic cache** — avoids redundant model calls for repeated or similar queries.
- **Prompt and guardrails engine** — enforces prompt structure, safety filters, and output validation.

### RAG Pipeline
- **Ingestion** — document loaders pull from enterprise sources, normalize content, and generate embeddings for indexing.
- **Vector database** — stores embeddings and supports hybrid search and semantic retrieval.
- **Retriever** — combines vector and keyword search with reranking before grounding the model response.

### Model Layer
- **Local / private LLMs** — self-hosted or fine-tuned models served on private GPU infrastructure for sensitive, regulated, or offline workloads.
- **External / managed LLMs** — managed providers used for general-purpose tasks requiring broad capability.

### Enterprise Data Sources
Structured databases, document repositories, and SaaS or internal APIs feed the ingestion and grounding pipeline.

### Observability, Governance & Ops
- **Logging and tracing** — end-to-end request visibility.
- **Evaluation and feedback** — quality scoring and human feedback capture.
- **Cost and usage monitoring** — spend, latency, and route selection by model.
- **Compliance and audit** — access records and decision traceability for regulated workloads.

## Routing Logic

| Condition | Preferred route |
|---|---|
| Sensitive or regulated data, offline requirement, or cost-sensitive high-volume workloads | Local / private LLM |
| General reasoning, broad capability, and lower data sensitivity | External / managed LLM |

## Final Notes

This architecture balances three enterprise priorities:

- Security and compliance
- Performance and cost control
- Quality and governability through RAG, evaluation, and telemetry

Separating ingestion, retrieval, orchestration, model selection, and monitoring keeps the platform modular, auditable, and scalable as AI workloads mature.
