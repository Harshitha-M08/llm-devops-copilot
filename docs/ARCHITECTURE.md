# AI-Driven Hybrid Kubernetes System - Architecture Documentation

## Table of Contents
1. [System Overview](#system-overview)
2. [Architecture Principles](#architecture-principles)
3. [High-Level Architecture](#high-level-architecture)
4. [Component Architecture](#component-architecture)
5. [Data Flow Diagrams](#data-flow-diagrams)
6. [Technology Stack](#technology-stack)
7. [Design Decisions](#design-decisions)
8. [Scalability Architecture](#scalability-architecture)
9. [High Availability](#high-availability)
10. [Security Architecture](#security-architecture)
11. [Network Architecture](#network-architecture)

---

## System Overview

The AI-Driven Hybrid Kubernetes System is a production-grade, cloud-native microservices platform designed to provide intelligent automation with human oversight. The system integrates multiple LLM providers (OpenAI, Anthropic), implements Retrieval-Augmented Generation (RAG) for contextual responses, and provides a comprehensive approval workflow for critical operations.

### Key Characteristics
- **Microservices Architecture**: Loosely coupled, independently deployable services
- **Event-Driven**: Asynchronous communication via message queues
- **Cloud-Native**: Designed for Kubernetes orchestration
- **Multi-Cloud Ready**: Deployable on AWS EKS, GCP GKE, or on-premises clusters
- **Horizontally Scalable**: Auto-scaling based on load metrics
- **Resilient**: Self-healing with health checks and automatic restarts
- **Observable**: Comprehensive monitoring, logging, and tracing

---

## Architecture Principles

### 1. Separation of Concerns
Each microservice has a single, well-defined responsibility:
- **LLM Service**: AI/ML operations and RAG queries
- **Worker Service**: Background task processing
- **Approval Backend**: Business logic and workflow management
- **Approval Frontend**: User interface and experience

### 2. API-First Design
All services expose RESTful APIs with clear contracts:
- OpenAPI/Swagger documentation
- Versioned endpoints (e.g., `/api/v1/...`)
- Standard HTTP status codes
- JSON request/response format

### 3. Asynchronous Communication
Long-running tasks use message queues:
- RabbitMQ for reliable message delivery
- Persistent message storage
- Dead-letter queues for failure handling
- Task acknowledgment and retry logic

### 4. Defense in Depth
Multiple security layers:
- Network policies for pod-to-pod communication
- RBAC for Kubernetes resources
- Secrets management with encryption
- TLS/SSL for all external traffic
- API authentication and authorization

### 5. Observability
Built-in monitoring and logging:
- Prometheus metrics for all services
- Grafana dashboards for visualization
- ELK stack for centralized logging
- Distributed tracing capability

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          External Layer                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────┐        ┌──────────────┐         ┌──────────────┐         │
│  │  End Users   │───────▶│   Ingress    │◀────────│    Admin     │         │
│  │  (Browser)   │        │   Gateway    │         │   Users      │         │
│  └──────────────┘        │  (NGINX)     │         └──────────────┘         │
│                          │  + TLS/SSL   │                                    │
│                          └──────┬───────┘                                    │
│                                 │                                            │
└─────────────────────────────────┼────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────┼────────────────────────────────────────────┐
│                          Application Layer                                   │
├─────────────────────────────────┼────────────────────────────────────────────┤
│                                 │                                            │
│    ┌────────────────────────────┴────────────────────────────┐              │
│    │                                                           │              │
│    ▼                                                           ▼              │
│  ┌──────────────────┐                            ┌─────────────────────┐    │
│  │  Approval        │                            │   LLM Service       │    │
│  │  Frontend        │◀──────────────────────────▶│   (FastAPI)         │    │
│  │  (React)         │      REST API              │                     │    │
│  │  Port: 80        │                            │   • Chat API        │    │
│  └──────────────────┘                            │   • RAG Pipeline    │    │
│           │                                       │   • Vector Store    │    │
│           │ WebSocket                             │   Port: 8000        │    │
│           ▼                                       └──────────┬──────────┘    │
│  ┌──────────────────┐                                       │                │
│  │  Approval        │                                       │                │
│  │  Backend         │                                       │                │
│  │  (Node.js)       │◀──────────────────────────────────────┘                │
│  │                  │           REST API                                     │
│  │  • Auth/RBAC     │                                                        │
│  │  • Workflow Mgmt │                                                        │
│  │  • Notifications │                                                        │
│  │  Port: 3000      │                                                        │
│  └─────────┬────────┘                                                        │
│            │                                                                  │
│            │ RabbitMQ                                                         │
│            ▼                                                                  │
│  ┌──────────────────┐                                                        │
│  │  Message Queue   │                                                        │
│  │  (RabbitMQ)      │                                                        │
│  │                  │                                                        │
│  │  • Task Queue    │                                                        │
│  │  • Event Bus     │                                                        │
│  │  • DLQ           │                                                        │
│  │  Port: 5672      │                                                        │
│  └─────────┬────────┘                                                        │
│            │                                                                  │
│            ▼                                                                  │
│  ┌──────────────────┐                                                        │
│  │  Worker Service  │                                                        │
│  │  (Python)        │                                                        │
│  │                  │                                                        │
│  │  • Task Consumer │                                                        │
│  │  • Async Ops     │                                                        │
│  │  • Schedulers    │                                                        │
│  └──────────────────┘                                                        │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────┼────────────────────────────────────────────┐
│                          Data Layer                                          │
├─────────────────────────────────┼────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────────┐       │
│  │   PostgreSQL     │  │      Redis       │  │   Qdrant Vector     │       │
│  │                  │  │                  │  │   Database          │       │
│  │  • User Data     │  │  • Cache         │  │                     │       │
│  │  • Approvals     │  │  • Sessions      │  │  • Embeddings       │       │
│  │  • Audit Logs    │  │  • Rate Limit    │  │  • Semantic Search  │       │
│  │  Port: 5432      │  │  Port: 6379      │  │  Port: 6333         │       │
│  └──────────────────┘  └──────────────────┘  └─────────────────────┘       │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────┼────────────────────────────────────────────┐
│                    External Services Layer                                   │
├─────────────────────────────────┼────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────────┐       │
│  │   OpenAI API     │  │  Anthropic API   │  │   Notification      │       │
│  │                  │  │                  │  │   Services          │       │
│  │  • GPT-4         │  │  • Claude        │  │                     │       │
│  │  • Embeddings    │  │  • Sonnet        │  │  • Email (SMTP)     │       │
│  │                  │  │                  │  │  • Slack            │       │
│  └──────────────────┘  └──────────────────┘  └─────────────────────┘       │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────┼────────────────────────────────────────────┐
│                    Observability Layer                                       │
├─────────────────────────────────┼────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────────┐       │
│  │   Prometheus     │  │     Grafana      │  │   ELK Stack         │       │
│  │                  │  │                  │  │                     │       │
│  │  • Metrics       │  │  • Dashboards    │  │  • Elasticsearch    │       │
│  │  • Alerting      │  │  • Visualization │  │  • Logstash         │       │
│  │  Port: 9090      │  │  Port: 3000      │  │  • Kibana           │       │
│  └──────────────────┘  └──────────────────┘  └─────────────────────┘       │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## Component Architecture

### 1. LLM Service (FastAPI)

**Purpose**: Provides AI/ML capabilities including chat completions, RAG queries, and document ingestion.

**Architecture**:
```
┌─────────────────────────────────────────────────┐
│            LLM Service Container                │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │         FastAPI Application              │  │
│  │  ┌────────────┐  ┌────────────────────┐  │  │
│  │  │   Routes   │  │   Request Models   │  │  │
│  │  │            │  │   (Pydantic)       │  │  │
│  │  │  /health   │  │                    │  │  │
│  │  │  /chat     │  │  • ChatRequest     │  │  │
│  │  │  /rag      │  │  • RAGRequest      │  │  │
│  │  │  /ingest   │  │  • DocRequest      │  │  │
│  │  └──────┬─────┘  └──────────┬─────────┘  │  │
│  │         │                   │             │  │
│  │         ▼                   ▼             │  │
│  │  ┌──────────────────────────────────┐    │  │
│  │  │      Business Logic Layer        │    │  │
│  │  │  ┌────────────┐  ┌─────────────┐ │    │  │
│  │  │  │ LLM Client │  │ RAG Pipeline│ │    │  │
│  │  │  │            │  │             │ │    │  │
│  │  │  │ • OpenAI   │  │ • Chunking  │ │    │  │
│  │  │  │ • Anthropic│  │ • Embedding │ │    │  │
│  │  │  │ • Failover │  │ • Retrieval │ │    │  │
│  │  │  └──────┬─────┘  └──────┬──────┘ │    │  │
│  │  └─────────┼────────────────┼────────┘    │  │
│  └────────────┼────────────────┼─────────────┘  │
│               │                │                 │
│               ▼                ▼                 │
│  ┌─────────────────────────────────────────┐    │
│  │      External Integrations              │    │
│  │  ┌────────────┐  ┌──────────────────┐   │    │
│  │  │ OpenAI SDK │  │ Qdrant Client    │   │    │
│  │  └────────────┘  └──────────────────┘   │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │      Middleware & Monitoring            │    │
│  │  • CORS                                 │    │
│  │  • Prometheus Metrics                   │    │
│  │  • Request Logging                      │    │
│  │  • Error Handling                       │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Key Features**:
- Multiple LLM provider support with automatic failover
- RAG pipeline with semantic search
- Document chunking and embedding generation
- Streaming response support
- Prometheus metrics exposure
- Health and readiness probes

**API Endpoints**:
- `GET /health` - Health check
- `GET /ready` - Readiness check
- `GET /metrics` - Prometheus metrics
- `POST /api/v1/chat` - Chat completion
- `POST /api/v1/chat/stream` - Streaming chat
- `POST /api/v1/documents/ingest` - Document ingestion
- `POST /api/v1/rag/query` - RAG query
- `POST /api/v1/rag/retrieve` - Document retrieval
- `POST /api/v1/tasks/submit` - Async task submission

### 2. Worker Service (Python)

**Purpose**: Processes background tasks asynchronously from the message queue.

**Architecture**:
```
┌─────────────────────────────────────────────────┐
│          Worker Service Container               │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │      RabbitMQ Consumer                   │  │
│  │  ┌────────────────────────────────────┐  │  │
│  │  │  Message Queue Listener            │  │  │
│  │  │  • Queue: llm_tasks                │  │  │
│  │  │  • Prefetch: 5                     │  │  │
│  │  │  • Auto-ack: False                 │  │  │
│  │  └──────────────┬─────────────────────┘  │  │
│  └─────────────────┼────────────────────────┘  │
│                    │                            │
│                    ▼                            │
│  ┌──────────────────────────────────────────┐  │
│  │         Task Dispatcher                  │  │
│  │  ┌────────────────────────────────────┐  │  │
│  │  │  Route by task_type:               │  │  │
│  │  │  • llm_completion                  │  │  │
│  │  │  • document_processing             │  │  │
│  │  │  • batch_operations                │  │  │
│  │  │  • scheduled_tasks                 │  │  │
│  │  └──────────────┬─────────────────────┘  │  │
│  └─────────────────┼────────────────────────┘  │
│                    │                            │
│                    ▼                            │
│  ┌──────────────────────────────────────────┐  │
│  │         Task Processors                  │  │
│  │  ┌────────────┐  ┌────────────────────┐  │  │
│  │  │   LLM      │  │   Data Processing  │  │  │
│  │  │   Tasks    │  │   Tasks            │  │  │
│  │  │            │  │                    │  │  │
│  │  └────────────┘  └────────────────────┘  │  │
│  │  ┌────────────┐  ┌────────────────────┐  │  │
│  │  │  Batch     │  │   Notification     │  │  │
│  │  │  Tasks     │  │   Tasks            │  │  │
│  │  └────────────┘  └────────────────────┘  │  │
│  └─────────────────────────────────────────┘  │
│                                                │
│  ┌─────────────────────────────────────────┐  │
│  │      Error Handling & Retry             │  │
│  │  • Retry logic with exponential backoff │  │
│  │  • Dead letter queue for failures       │  │
│  │  • Task timeout management              │  │
│  └─────────────────────────────────────────┘  │
│                                                │
└────────────────────────────────────────────────┘
```

**Key Features**:
- Asynchronous task processing
- Multiple worker threads/processes
- Retry logic with exponential backoff
- Dead letter queue for failed tasks
- Task timeout management
- Health monitoring

### 3. Approval Backend (Node.js/Express)

**Purpose**: Manages approval workflows, authentication, and business logic.

**Architecture**:
```
┌─────────────────────────────────────────────────┐
│        Approval Backend Container               │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │      Express.js Application              │  │
│  │  ┌────────────────────────────────────┐  │  │
│  │  │        Middleware Stack            │  │  │
│  │  │  • CORS                            │  │  │
│  │  │  • Body Parser                     │  │  │
│  │  │  • JWT Authentication              │  │  │
│  │  │  • Request Logging                 │  │  │
│  │  │  • Error Handler                   │  │  │
│  │  └──────────────┬─────────────────────┘  │  │
│  └─────────────────┼────────────────────────┘  │
│                    │                            │
│                    ▼                            │
│  ┌──────────────────────────────────────────┐  │
│  │         API Routes                       │  │
│  │  ┌────────────┐  ┌────────────────────┐  │  │
│  │  │   Auth     │  │   Approvals        │  │  │
│  │  │  /login    │  │  /approvals        │  │  │
│  │  │  /logout   │  │  /approvals/:id    │  │  │
│  │  │  /refresh  │  │  /approve          │  │  │
│  │  └────────────┘  └────────────────────┘  │  │
│  │  ┌────────────┐  ┌────────────────────┐  │  │
│  │  │   Users    │  │   Notifications    │  │  │
│  │  │  /users    │  │  /notifications    │  │  │
│  │  │  /profile  │  │  /subscribe        │  │  │
│  │  └────────────┘  └────────────────────┘  │  │
│  └─────────────────────────────────────────┘  │
│                    │                            │
│                    ▼                            │
│  ┌──────────────────────────────────────────┐  │
│  │      Business Logic Layer                │  │
│  │  ┌────────────┐  ┌────────────────────┐  │  │
│  │  │ Approval   │  │  Notification      │  │  │
│  │  │ Service    │  │  Service           │  │  │
│  │  │            │  │                    │  │  │
│  │  │ • Workflow │  │  • Email           │  │  │
│  │  │ • RBAC     │  │  • WebSocket       │  │  │
│  │  │ • Audit    │  │  • Slack           │  │  │
│  │  └────────────┘  └────────────────────┘  │  │
│  └─────────────────────────────────────────┘  │
│                    │                            │
│                    ▼                            │
│  ┌──────────────────────────────────────────┐  │
│  │      Data Access Layer                   │  │
│  │  ┌────────────┐  ┌────────────────────┐  │  │
│  │  │ PostgreSQL │  │      Redis         │  │  │
│  │  │  Client    │  │      Client        │  │  │
│  │  │            │  │                    │  │  │
│  │  │ • ORM      │  │  • Cache           │  │  │
│  │  │ • Queries  │  │  • Sessions        │  │  │
│  │  └────────────┘  └────────────────────┘  │  │
│  └─────────────────────────────────────────┘  │
│                                                │
│  ┌─────────────────────────────────────────┐  │
│  │      WebSocket Server                    │  │
│  │  • Real-time notifications              │  │
│  │  • Connection management                │  │
│  │  • Event broadcasting                   │  │
│  └─────────────────────────────────────────┘  │
│                                                │
└────────────────────────────────────────────────┘
```

**Key Features**:
- JWT-based authentication
- Role-based access control (RBAC)
- Approval workflow management
- Real-time notifications via WebSocket
- Audit logging
- Email notifications

### 4. Approval Frontend (React)

**Purpose**: User interface for managing approvals and system interactions.

**Architecture**:
```
┌─────────────────────────────────────────────────┐
│        Approval Frontend Container              │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │         React Application                │  │
│  │  ┌────────────────────────────────────┐  │  │
│  │  │       Component Tree               │  │  │
│  │  │                                    │  │  │
│  │  │  App                               │  │  │
│  │  │   ├── Router                       │  │  │
│  │  │   ├── Header                       │  │  │
│  │  │   ├── Sidebar                      │  │  │
│  │  │   └── Routes                       │  │  │
│  │  │       ├── Login                    │  │  │
│  │  │       ├── Dashboard                │  │  │
│  │  │       ├── Approvals                │  │  │
│  │  │       │   ├── ApprovalList         │  │  │
│  │  │       │   └── ApprovalDetail       │  │  │
│  │  │       ├── Users                    │  │  │
│  │  │       └── Settings                 │  │  │
│  │  └────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────┘  │
│                    │                            │
│                    ▼                            │
│  ┌──────────────────────────────────────────┐  │
│  │       State Management (Redux)           │  │
│  │  ┌────────────┐  ┌────────────────────┐  │  │
│  │  │   Store    │  │     Actions        │  │  │
│  │  │            │  │                    │  │  │
│  │  │ • Auth     │  │  • Login           │  │  │
│  │  │ • Approvals│  │  • Fetch Data      │  │  │
│  │  │ • Users    │  │  • Submit          │  │  │
│  │  │ • Notifs   │  │  • Update          │  │  │
│  │  └────────────┘  └────────────────────┘  │  │
│  └─────────────────────────────────────────┘  │
│                    │                            │
│                    ▼                            │
│  ┌──────────────────────────────────────────┐  │
│  │         API Service Layer                │  │
│  │  ┌────────────┐  ┌────────────────────┐  │  │
│  │  │   HTTP     │  │   WebSocket        │  │  │
│  │  │   Client   │  │   Client           │  │  │
│  │  │            │  │                    │  │  │
│  │  │ • Axios    │  │  • Socket.io       │  │  │
│  │  │ • Auth     │  │  • Events          │  │  │
│  │  │ • Retry    │  │  • Reconnect       │  │  │
│  │  └────────────┘  └────────────────────┘  │  │
│  └─────────────────────────────────────────┘  │
│                                                │
│  ┌─────────────────────────────────────────┐  │
│  │         Utility Components               │  │
│  │  • Notifications                        │  │
│  │  • Loading Spinners                     │  │
│  │  • Error Boundaries                     │  │
│  │  • Form Validation                      │  │
│  └─────────────────────────────────────────┘  │
│                                                │
└────────────────────────────────────────────────┘
```

**Key Features**:
- Single Page Application (SPA)
- Real-time updates via WebSocket
- Responsive design
- Role-based UI rendering
- Form validation
- Error handling

---

## Data Flow Diagrams

### 1. Chat Completion Flow

```
┌─────────┐         ┌─────────────┐         ┌─────────────┐         ┌─────────┐
│  User   │────────▶│   Ingress   │────────▶│ LLM Service │────────▶│ OpenAI  │
│         │  HTTPS  │   Gateway   │  HTTP   │             │  HTTPS  │   API   │
└─────────┘         └─────────────┘         └──────┬──────┘         └─────────┘
                                                    │
                                                    │ Cache Check
                                                    ▼
                                             ┌─────────────┐
                                             │    Redis    │
                                             │   (Cache)   │
                                             └─────────────┘
                                                    │
                                                    │ Response
                                                    ▼
┌─────────┐         ┌─────────────┐         ┌─────────────┐
│  User   │◀────────│   Ingress   │◀────────│ LLM Service │
│         │  HTTPS  │   Gateway   │  HTTP   │             │
└─────────┘         └─────────────┘         └─────────────┘
```

### 2. RAG Query Flow

```
┌─────────┐
│  User   │
│ submits │
│  query  │
└────┬────┘
     │
     ▼
┌─────────────┐      1. Receive Query
│ LLM Service │────────────────────┐
└──────┬──────┘                    │
       │                           │
       │ 2. Generate Embedding     │
       ▼                           │
┌─────────────┐                    │
│   OpenAI    │                    │
│ Embeddings  │                    │
└──────┬──────┘                    │
       │                           │
       │ 3. Vector Search          ▼
       ▼                    ┌─────────────┐
┌─────────────┐            │   Qdrant    │
│   Qdrant    │◀───────────│   Vector    │
│   Vector    │            │   Database  │
│   Database  │            └─────────────┘
└──────┬──────┘
       │
       │ 4. Retrieve Top-K Documents
       ▼
┌─────────────┐
│ LLM Service │
│ (Context +  │
│  Query)     │
└──────┬──────┘
       │
       │ 5. Send to LLM with Context
       ▼
┌─────────────┐
│   OpenAI    │
│   GPT-4     │
└──────┬──────┘
       │
       │ 6. Generate Answer
       ▼
┌─────────────┐
│  Response   │
│  to User    │
└─────────────┘
```

### 3. Approval Workflow Flow

```
┌─────────┐         ┌──────────────┐         ┌──────────────┐
│ System  │────────▶│   Approval   │────────▶│  PostgreSQL  │
│ Action  │  1.     │   Backend    │  2.     │   Database   │
│ Requires│ Create  │              │ Store   │              │
│Approval │ Request └──────┬───────┘         └──────────────┘
└─────────┘                │
                           │ 3. Notify
                           ▼
                    ┌──────────────┐
                    │ Notification │
                    │   Service    │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
       ┌──────────┐ ┌──────────┐ ┌──────────┐
       │WebSocket │ │  Email   │ │  Slack   │
       │  to UI   │ │  Alert   │ │  Message │
       └────┬─────┘ └──────────┘ └──────────┘
            │
            ▼
       ┌──────────┐
       │ Approver │
       │  Views   │
       │  in UI   │
       └────┬─────┘
            │
            │ 4. Decision (Approve/Deny)
            ▼
       ┌──────────────┐
       │   Approval   │
       │   Backend    │
       └──────┬───────┘
              │
              │ 5. Update Status
              ▼
       ┌──────────────┐         ┌──────────────┐
       │  PostgreSQL  │         │   RabbitMQ   │
       │   Database   │         │    Queue     │
       └──────────────┘         └──────┬───────┘
                                       │
                                       │ 6. Execute Action
                                       ▼
                                ┌──────────────┐
                                │   Worker     │
                                │   Service    │
                                └──────────────┘
```

### 4. Async Task Processing Flow

```
┌─────────────┐    1. Submit    ┌─────────────┐    2. Publish    ┌─────────────┐
│ LLM Service │────────────────▶│   RabbitMQ  │─────────────────▶│   Worker    │
│             │     Task        │    Queue    │    Message       │   Service   │
└─────────────┘                 └─────────────┘                  └──────┬──────┘
      │                                                                  │
      │ 3. Return Task ID                                                │
      ▼                                                                  │
┌─────────────┐                                                          │
│   Client    │                                                          │
│  (Polling)  │                                                          │
└─────────────┘                                                          │
                                                                         │
                                                         4. Process Task │
                                                                         ▼
                                                                  ┌─────────────┐
                                                                  │ LLM Service │
                                                                  │     or      │
                                                                  │  External   │
                                                                  │   Service   │
                                                                  └──────┬──────┘
                                                                         │
                                                         5. Store Result │
                                                                         ▼
┌─────────────┐    6. Query     ┌─────────────┐                  ┌─────────────┐
│   Client    │────────────────▶│   Approval  │◀─────────────────│  PostgreSQL │
│             │     Status      │   Backend   │    7. Fetch      │   Database  │
└─────────────┘                 └─────────────┘                  └─────────────┘
```

---

## Technology Stack

### Application Layer

#### LLM Service
- **Framework**: FastAPI 0.104+
- **Language**: Python 3.11+
- **Libraries**:
  - `openai` - OpenAI API client
  - `anthropic` - Anthropic API client
  - `langchain` - LLM orchestration
  - `qdrant-client` - Vector database client
  - `pydantic` - Data validation
  - `prometheus-client` - Metrics
  - `pika` - RabbitMQ client
  - `uvicorn` - ASGI server

#### Worker Service
- **Framework**: Custom Python worker
- **Language**: Python 3.11+
- **Libraries**:
  - `pika` - RabbitMQ consumer
  - `celery` - Task queue (optional)
  - `schedule` - Task scheduling
  - `requests` - HTTP client

#### Approval Backend
- **Framework**: Express.js 4.18+
- **Runtime**: Node.js 18 LTS
- **Libraries**:
  - `express` - Web framework
  - `jsonwebtoken` - JWT authentication
  - `bcrypt` - Password hashing
  - `socket.io` - WebSocket
  - `pg` - PostgreSQL client
  - `redis` - Redis client
  - `nodemailer` - Email sending

#### Approval Frontend
- **Framework**: React 18+
- **Language**: JavaScript/TypeScript
- **Libraries**:
  - `react-router-dom` - Routing
  - `redux` / `zustand` - State management
  - `axios` - HTTP client
  - `socket.io-client` - WebSocket
  - `tailwindcss` - Styling
  - `react-query` - Data fetching

### Data Layer

#### PostgreSQL
- **Version**: 15+
- **Purpose**: Relational data storage
- **Stores**: Users, approvals, audit logs, configurations

#### Redis
- **Version**: 7+
- **Purpose**: Caching and session storage
- **Uses**: API response cache, user sessions, rate limiting

#### Qdrant
- **Version**: 1.7+
- **Purpose**: Vector database for embeddings
- **Uses**: Semantic search, RAG retrieval

### Infrastructure Layer

#### Kubernetes
- **Version**: 1.24+
- **Components**:
  - Deployments
  - Services (ClusterIP, LoadBalancer)
  - StatefulSets (databases)
  - ConfigMaps
  - Secrets
  - Ingress
  - HPA (Horizontal Pod Autoscaler)
  - Network Policies
  - RBAC

#### Message Queue
- **Service**: RabbitMQ 3.12+
- **Features**: Persistent queues, dead letter queues, message acknowledgment

#### Monitoring
- **Prometheus**: Metrics collection
- **Grafana**: Visualization and dashboards
- **Elasticsearch**: Log aggregation
- **Logstash**: Log processing
- **Kibana**: Log visualization
- **Alertmanager**: Alert routing and notification

---

## Design Decisions

### 1. Why Microservices?

**Decision**: Use microservices architecture instead of monolith.

**Rationale**:
- **Independent Scaling**: LLM service can scale independently based on AI workload
- **Technology Flexibility**: Python for AI/ML, Node.js for real-time features
- **Team Independence**: Different teams can work on different services
- **Fault Isolation**: Failure in one service doesn't bring down the entire system
- **Deployment Flexibility**: Services can be updated independently

**Trade-offs**:
- Increased complexity in deployment and orchestration
- Network latency between services
- Distributed debugging challenges
- Need for service mesh or API gateway

### 2. Why RabbitMQ over Kafka?

**Decision**: Use RabbitMQ for message queueing.

**Rationale**:
- **Simplicity**: Easier to set up and manage
- **Message Routing**: Advanced routing capabilities with exchanges
- **Lower Resource Usage**: Less memory and CPU overhead
- **Good Enough**: Throughput requirements don't justify Kafka complexity
- **Community Support**: Large ecosystem and documentation

**Trade-offs**:
- Lower throughput compared to Kafka
- Less suitable for streaming use cases
- Limited distributed log capabilities

### 3. Why Qdrant over Pinecone/Weaviate?

**Decision**: Use Qdrant as vector database.

**Rationale**:
- **Performance**: Fast vector similarity search
- **Self-Hosted Option**: Can be deployed in Kubernetes
- **API Simplicity**: Clean REST and gRPC APIs
- **Filtering**: Advanced payload filtering capabilities
- **Free Tier**: Generous free tier for cloud version

**Trade-offs**:
- Smaller community compared to Pinecone
- Less mature than some alternatives
- Limited managed service options

### 4. Why FastAPI over Flask?

**Decision**: Use FastAPI for LLM service.

**Rationale**:
- **Performance**: Async/await support for concurrent requests
- **Type Safety**: Pydantic models with automatic validation
- **Auto Documentation**: Built-in Swagger UI
- **Modern**: Latest Python features and best practices
- **Streaming**: Native support for streaming responses

**Trade-offs**:
- Steeper learning curve than Flask
- Smaller ecosystem of plugins
- Requires Python 3.7+

### 5. Why JWT for Authentication?

**Decision**: Use JWT tokens for API authentication.

**Rationale**:
- **Stateless**: No server-side session storage needed
- **Scalable**: Works well with load balancing
- **Standard**: Well-understood and widely supported
- **Flexible**: Can include custom claims
- **Mobile-Friendly**: Easy to use in mobile apps

**Trade-offs**:
- Cannot invalidate tokens before expiry
- Token size can be large
- Need refresh token mechanism

### 6. Why PostgreSQL over MongoDB?

**Decision**: Use PostgreSQL for relational data.

**Rationale**:
- **ACID Compliance**: Strong consistency guarantees
- **Relational Model**: Approval workflows are inherently relational
- **Mature**: Battle-tested with excellent tooling
- **JSON Support**: Can store JSON data when needed
- **Full-Text Search**: Built-in search capabilities

**Trade-offs**:
- Less flexible schema than NoSQL
- Vertical scaling limitations
- More complex sharding

---

## Scalability Architecture

### Horizontal Scaling Strategy

#### 1. Application Layer Scaling

**LLM Service**:
- **Min Replicas**: 3
- **Max Replicas**: 10
- **Scaling Metrics**: CPU (70%), Memory (80%)
- **Strategy**: Scale up quickly, scale down conservatively

```yaml
HPA Configuration:
  scaleUp:
    - 100% pods every 30 seconds
    - Max 2 pods per 30 seconds
  scaleDown:
    - 50% pods every 60 seconds
    - Max 1 pod per 60 seconds
    - 5 minute stabilization window
```

**Worker Service**:
- **Min Replicas**: 3
- **Max Replicas**: 15
- **Scaling Metrics**: RabbitMQ queue depth
- **Strategy**: KEDA-based event-driven scaling

```yaml
KEDA ScaledObject:
  trigger: rabbitmq
  queueName: llm_tasks
  queueLength: 10 messages per pod
  cooldownPeriod: 300 seconds
```

**Approval Backend**:
- **Min Replicas**: 3
- **Max Replicas**: 10
- **Scaling Metrics**: CPU (70%), Request rate
- **Strategy**: Standard HPA with custom metrics

**Approval Frontend**:
- **Min Replicas**: 2
- **Max Replicas**: 8
- **Scaling Metrics**: CPU (70%)
- **Strategy**: Standard HPA

#### 2. Data Layer Scaling

**PostgreSQL**:
- **Vertical Scaling**: Increase pod resources (CPU/memory)
- **Read Replicas**: 2-3 read replicas for read-heavy workloads
- **Connection Pooling**: PgBouncer for connection management
- **Sharding**: Partition by tenant_id if needed

**Redis**:
- **Sentinel Mode**: High availability with automatic failover
- **Cluster Mode**: Sharding for large datasets
- **Vertical Scaling**: Increase memory allocation

**Qdrant**:
- **Managed Service**: Use Qdrant Cloud with auto-scaling
- **Self-Hosted**: Increase replicas and use distributed mode

#### 3. Load Distribution

**Ingress Load Balancing**:
```
Traffic Distribution:
  Algorithm: Round-robin with session affinity
  Health Checks: HTTP /health endpoint every 10s
  Connection Timeouts: 60s
  Max Connections per Pod: 1000
```

**Service Mesh** (Optional):
```
Istio/Linkerd:
  - Circuit breaking
  - Retry policies
  - Traffic splitting (A/B testing)
  - mTLS between services
```

### Vertical Scaling Guidelines

**When to Scale Vertically**:
1. Consistent high CPU/memory usage across all pods
2. Database performance bottlenecks
3. Individual request processing time increases
4. Before major traffic events

**Resource Allocation**:
```yaml
LLM Service:
  requests:
    memory: 2Gi
    cpu: 1000m
  limits:
    memory: 4Gi
    cpu: 2000m

Worker Service:
  requests:
    memory: 1Gi
    cpu: 500m
  limits:
    memory: 2Gi
    cpu: 1000m

Approval Backend:
  requests:
    memory: 512Mi
    cpu: 250m
  limits:
    memory: 1Gi
    cpu: 500m
```

### Caching Strategy

**Redis Caching**:
```
Cache Layers:
  L1: LLM responses (TTL: 1 hour)
  L2: Database queries (TTL: 5 minutes)
  L3: API responses (TTL: 30 seconds)

Eviction Policy:
  - LRU (Least Recently Used)
  - Max memory: 2GB per Redis instance
```

**Application-Level Caching**:
- In-memory cache for configuration
- Memoization for expensive computations
- CDN for static assets (frontend)

---

## High Availability

### Redundancy Strategy

#### 1. Application Redundancy

**Multi-Replica Deployment**:
- Minimum 3 replicas per service
- Pods distributed across availability zones
- Anti-affinity rules prevent co-location

```yaml
Pod Anti-Affinity:
  preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values: [llm-service]
        topologyKey: kubernetes.io/hostname
```

#### 2. Data Redundancy

**PostgreSQL**:
- Primary-replica replication
- Automatic failover with Patroni or Stolon
- Continuous archiving with WAL
- Point-in-time recovery capability

**Redis**:
- Sentinel mode with 3 sentinels
- Automatic failover in <30 seconds
- Persistence with AOF and RDB snapshots

**RabbitMQ**:
- Clustered deployment with 3 nodes
- Mirrored queues for critical data
- Persistent messages on disk

#### 3. Network Redundancy

**Ingress**:
- Multiple ingress controllers
- External load balancer with health checks
- DNS-based failover

**Service Mesh** (Optional):
- Automatic retries on failure
- Circuit breaker pattern
- Fallback responses

### Health Checks

**Kubernetes Probes**:

```yaml
Liveness Probe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

Readiness Probe:
  httpGet:
    path: /ready
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3

Startup Probe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 30
```

**Health Check Endpoints**:
- `/health` - Basic service health
- `/ready` - Service ready to accept traffic
- `/metrics` - Prometheus metrics

### Disaster Recovery

**Backup Strategy**:
```
Daily Backups:
  - PostgreSQL: Full backup + WAL archiving
  - Redis: RDB snapshots
  - Configuration: Git repository
  - Secrets: External secret manager

Retention:
  - Daily backups: 7 days
  - Weekly backups: 4 weeks
  - Monthly backups: 12 months
```

**Recovery Time Objectives**:
- **RTO** (Recovery Time Objective): 1 hour
- **RPO** (Recovery Point Objective): 5 minutes

**Disaster Recovery Procedures**:
1. Switch to backup cluster (if available)
2. Restore database from latest backup
3. Redeploy services from container registry
4. Update DNS to point to new cluster
5. Verify system functionality

---

## Security Architecture

### Defense-in-Depth Layers

#### Layer 1: Network Security

**Kubernetes Network Policies**:
```yaml
Default Deny All:
  - Block all ingress and egress by default
  - Explicit allow rules for each communication path

Allowed Connections:
  - Frontend -> Backend: TCP/3000
  - Backend -> Database: TCP/5432
  - Backend -> Redis: TCP/6379
  - All Services -> RabbitMQ: TCP/5672
  - LLM Service -> External APIs: HTTPS/443
  - All Services -> DNS: UDP/53
```

**Ingress Security**:
- TLS 1.3 only
- Strong cipher suites
- HSTS headers
- Rate limiting (100 req/sec per IP)
- Connection limits (10 concurrent per IP)

#### Layer 2: Authentication & Authorization

**JWT Authentication**:
```
Token Structure:
  - Header: Algorithm (RS256)
  - Payload: User ID, roles, expiry
  - Signature: RSA private key

Token Lifecycle:
  - Access Token TTL: 15 minutes
  - Refresh Token TTL: 7 days
  - Rotation on refresh
```

**RBAC Roles**:
- **Admin**: Full system access
- **Approver**: Can approve/deny requests
- **Viewer**: Read-only access
- **Service**: Service-to-service communication

#### Layer 3: API Security

**Request Validation**:
- Schema validation with Pydantic/Joi
- Input sanitization
- SQL injection prevention
- XSS protection

**Rate Limiting**:
```
Per-User Limits:
  - API calls: 1000/hour
  - LLM requests: 100/hour
  - Document uploads: 10/hour

Per-IP Limits:
  - Anonymous: 100/hour
  - Authenticated: 1000/hour
```

#### Layer 4: Data Security

**Encryption**:
- **At Rest**: Database encryption, encrypted volumes
- **In Transit**: TLS for all communications
- **Secrets**: Kubernetes Secrets with encryption at rest

**Secret Management**:
```
Strategy:
  - Development: Kubernetes Secrets
  - Production: External Secrets Operator
    - AWS Secrets Manager
    - HashiCorp Vault
    - Google Secret Manager

Rotation:
  - Database passwords: 90 days
  - API keys: 180 days
  - JWT secrets: 365 days
```

#### Layer 5: Application Security

**Security Headers**:
```http
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
Content-Security-Policy: default-src 'self'
Strict-Transport-Security: max-age=31536000
```

**Container Security**:
```yaml
Security Context:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]
```

### Security Monitoring

**Audit Logging**:
- All authentication attempts
- All approval actions
- Configuration changes
- Failed authorization attempts
- Suspicious activity patterns

**Vulnerability Scanning**:
- Container image scanning in CI/CD
- Dependency vulnerability scanning
- Regular penetration testing
- Security advisories monitoring

---

## Network Architecture

### Internal Networking

**Kubernetes Networking**:
```
Pod Network:
  CIDR: 10.244.0.0/16
  CNI: Calico/Cilium

Service Network:
  CIDR: 10.96.0.0/12
  Type: ClusterIP (internal)
        LoadBalancer (external)

DNS:
  CoreDNS for service discovery
  Format: <service>.<namespace>.svc.cluster.local
```

**Service Communication**:
```
Internal Services (ClusterIP):
  - llm-service:8000
  - approval-backend:3000
  - postgres:5432
  - redis:6379
  - rabbitmq:5672

External Services (LoadBalancer):
  - approval-frontend:80/443
  - ingress-nginx:80/443
```

### External Networking

**Ingress Configuration**:
```yaml
Ingress Rules:
  - ai-system.example.com -> approval-frontend:80
  - api.ai-system.example.com/api -> approval-backend:3000
  - api.ai-system.example.com/llm -> llm-service:8000

TLS Termination:
  - Certificate: Let's Encrypt
  - Auto-renewal via cert-manager
  - Redirect HTTP to HTTPS
```

**External Dependencies**:
```
Egress Traffic:
  - OpenAI API: api.openai.com:443
  - Anthropic API: api.anthropic.com:443
  - Qdrant Cloud: *.qdrant.io:443
  - SMTP Server: smtp.gmail.com:587
  - Slack API: slack.com:443
```

### Network Security Zones

```
┌─────────────────────────────────────────────────┐
│             Public Zone (Internet)              │
│  • Web browsers                                 │
│  • External APIs                                │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│             DMZ (Ingress Layer)                 │
│  • Load Balancer                                │
│  • Ingress Controller                           │
│  • TLS Termination                              │
│  • WAF (Optional)                               │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│          Application Zone (Frontend)            │
│  • Approval Frontend                            │
│  • Static Assets                                │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│          Application Zone (Backend)             │
│  • LLM Service                                  │
│  • Approval Backend                             │
│  • Worker Service                               │
│  • RabbitMQ                                     │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│              Data Zone (Databases)              │
│  • PostgreSQL                                   │
│  • Redis                                        │
│  • Qdrant                                       │
│  • Backup Storage                               │
└─────────────────────────────────────────────────┘
```

**Zone Isolation**:
- Network policies enforce zone boundaries
- No direct access from public to data zone
- Application zone is the only bridge
- Monitoring zone has read-only access to all zones

---

## Conclusion

This architecture provides a robust, scalable, and secure foundation for an AI-driven system with human oversight. The microservices approach allows for independent scaling and deployment, while the comprehensive security measures ensure data protection and system integrity.

Key strengths:
- Horizontally scalable at all layers
- High availability with redundancy
- Defense-in-depth security
- Observable and monitorable
- Cloud-agnostic design

The architecture is designed to evolve with changing requirements while maintaining stability and reliability.
