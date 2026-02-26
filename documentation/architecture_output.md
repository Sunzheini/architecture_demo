# Prompt:
I will be team lead in a company that will develop a new software product, we don't have software 
engineers yet and no project software, repository, anything. We have to make them on the basis of 
best practices. First now we search for an architect to make an architecture of the software based 
on our requirements, that we have. The requirements are comprised of what modules we want (18 in 
total) and what each module will do. We want to use AI agents in each module where applicable. Now 
I will ask you questions, and you will answer it to me as concise as possible. What must be the 
output of the architect's work.


# Architect output:
## High-Level System Design: Diagram and description of overall structure, showing how the 18 modules interact.
A block diagram showing your 18 modules as boxes, with lines connecting them to illustrate 
communication. It defines the system boundaries, major components, and the interactions between 
them.
https://c4model.com/diagrams/system-context


## Technology Stack Recommendation: Specific languages, frameworks, and databases justified by your requirements.
A table or a layered diagram:
Layer	                        Technology Choice	                Alternatives Considered	        Primary Justification (Why this choice?)
Backend (Core)	                Python (FastAPI)	                Java (Spring Boot), C# (.NET)	Speed of development for 18 modules; best-in-class AI/ML library support (LangChain, PyTorch) which is critical for your AI agents.
AI/Agent                        Framework	LangChain / LangGraph	CrewAI, AutoGen	                Industry standard for agent orchestration; supports complex multi-agent workflows across your 18 modules.
Database (Operational)	        PostgreSQL	                        MySQL, MongoDB	                Reliability and extensions. Supports JSON for flexibility and relational data; crucial pgvector extension for AI embedding storage.
Database (Vector/Agent Memory)	Pinecone / pgvector	                Weaviate, Qdrant	            Specifically required for long-term memory for your AI agents; high-performance similarity search.
Frontend	                    React (TypeScript)	                Vue.js, Svelte	                Large ecosystem; component reusability for potentially different UIs across modules; strong typing with TypeScript prevents errors .
Infrastructure	                Docker + Kubernetes	                AWS ECS, Google Cloud Run	    Scalability and consistency. Ensures all 18 modules can be deployed and scaled independently and uniformly.


## AI Integration Strategy: A blueprint for where and how AI agents will be embedded in each applicable module (e.g., agent orchestration, model hosting, data flow).
A blueprint detailing how AI agents will be built, hosted, and invoked within the modules. Also 
prevents the 18 AI agents from becoming chaotic, unmanageable, or conflicting with each other. It 
defines the "rules of engagement" for all AI components.

Visual: An Orchestration Diagram. This shows how a user request triggers an agent. For example: 
User Input -> Supervisor Agent -> [Module A Agent | Module B Agent] -> Response.

"Agent Topology" Diagram
https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns

"Agent Definition" Table
Module	    Agent Name	    Primary Responsibility	Allowed Tools/Actions	                Data Access Scope
Module 1	SupportAgent	Answer customer FAQs	Search Knowledge Base, Create Ticket	Read: KB articles, User profile
Module 2	AnalyticsAgent	Analyze sales data	    Run SQL queries, Generate charts	    Read: Sales DB (Read-only)
Module 3	ContentAgent	Generate marketing copy	Call OpenAI, Access Template Library	Read: Brand guidelines; Write: Drafts folder


## Data Architecture: ER diagrams and specifications for data storage and flow between modules.
The design of your databases and how data moves between them. Crucial for 18 modules to share a 
single source of truth.

An Entity-Relationship Diagram (ERD) . Tables (entities) with columns (attributes) connected by lines showing relationships (one-to-many, many-to-many).
https://www.lucidchart.com/pages/er-diagrams

A Data Flow Diagram (DFD) . Shows where data originates, how it transforms, and where it is stored.
https://www.visual-paradigm.com/tutorials/data-flow-diagram-example-cs-system.jsp

Data dictionary defining every data field.


## Infrastructure Design: Cloud vs. on-premise decision, deployment diagrams, and scalability plans.
A blueprint for the physical or cloud environment where your software will live and run. It answers: 
"Where will this be deployed, and how will it handle traffic, failures, and scaling?"

Deployment Diagram
                                   [Internet]
                                        |
                                    [Firewall]
                                        |
                                    [Load Balancer]
                                   (SSL Termination)
                                        |
                +-----------------------+-----------------------+
                |                       |                       |
        [Web Server Fleet]       [Application Server Fleet]   [AI Agent Server Fleet]
        (Nginx/Apache)           (Python/Node.js Pods)        (GPU Instances)
        - Serves static files     - Runs Module Logic          - Runs LangChain/LangGraph
        - Reverse proxy           - 5-10 instances             - 2-3 instances (auto-scaled)
                |                       |                       |
                +-----------------------+-----------------------+
                                        |
                            [Internal Network (VPC)]
                                        |
            +---------------------------+---------------------------+
            |                           |                           |
    [Primary Database]          [Replica Database]           [Vector Database]
    (PostgreSQL)                (Read-only copy)             (Pinecone/Weaviate)
    - Writes                     - Reports/Analytics          - AI Agent Memory
    - Transactions                                          - Embeddings

The Infrastructure Specification
Component	            Technology	                    Specification / Configuration	                        Justification
Compute (Orchestration)	Kubernetes (EKS/AKS/GKE)	    3-5 nodes per environment (dev/staging/prod)	        Standardizes deployment for 18 modules; auto-healing and scaling.
Compute (AI/GPU)	    Dedicated GPU Nodes	            Cloud provider GPU instances (e.g., AWS G4dn or P3)	    Required for running/fine-tuning local LLMs (if not using OpenAI API).
Database (SQL)	        PostgreSQL (RDS/Azure DB)	    Multi-AZ deployment; Provisioned IOPS	                High availability; critical for transactional data across modules.
Database (Vector)	    Pinecone / Weaviate Cloud	    Standard pod tier with autoscaling	                    Purpose-built for AI embedding storage; offloads vector maintenance.
Cache	                Redis (ElastiCache)	            Clustered mode	                                        Session storage; rate-limiting; caching AI responses to reduce costs.
Message Queue	        RabbitMQ / Kafka (Confluent)	Clustered (3+ brokers)	                                Decouples module communication; essential for async AI agent tasks.
Storage	                S3 / Azure Blob / GCS	        Standard tier	                                        Storing user uploads, files, model artifacts.
CDN	                    CloudFront / Cloudflare	        Global edge network	                                    Low-latency delivery of static assets and UI.


## API Contract / Interface Definitions: Clear specifications for how modules communicate (REST, GraphQL, message queues).
The technical "handshake" agreement between modules. Since different teams might build 
different modules, this defines how they talk to each other.

OpenAPI/Swagger Specification
SWAGGER UI (http://localhost:8000/docs)
============================================

MODULE 1 API - Support Agent
--------------------------------------------------------------------------------
GET    /api/v1/tickets           → List all support tickets
POST   /api/v1/tickets           → Create a new ticket
GET    /api/v1/tickets/{id}      → Get ticket details
POST   /api/v1/ai/classify       → AI agent: classify ticket priority

[Try it out] [Execute]

Request Body (for POST /tickets):
{
  "customer_id": "string",
  "subject": "string",
  "description": "string",
  "priority": "low|medium|high"
}

Response (200 OK):
{
  "ticket_id": "uuid",
  "status": "created",
  "ai_suggested_category": "billing_issue",
  "estimated_response_time": "5 minutes"
}
--------------------------------------------------------------------------------

AsyncAPI Specification (For Event-Driven Communication)
AsyncAPI Document (Module Events)
==================================
Channels:
  └── module.ticket.created
      ├── Description: "Emitted when a new support ticket is created"
      ├── Content Type: application/json
      └── Payload:
          {
            "ticket_id": "uuid",
            "customer_id": "string",
            "created_at": "timestamp",
            "initial_ai_analysis": "object"
          }

Publishers:
  └── Module 1 (Support) → Publishes to this channel

Subscribers:
  └── Module 5 (Analytics) → Listens for reporting
  └── Module 9 (Notification) → Listens to email customer

API Style Guide
Naming Convention, Error Handling, Authentication, Versioning, etc.


## Technical Roadmap: Prioritized breakdown of which modules to build first based on technical dependencies.
A phased plan for building the software based on the architecture
Gantt chart or Timeline view
    Phase 1 (MVP): Core Module 1, Module 2, Database Setup, Authentication.
    Phase 2 (AI Integration): Implement AI agent in Module 3, Setup Vector DB.
    Phase 3: Remaining modules.
    Text: Identification of "critical path" dependencies (e.g., "Module 5 cannot start until the Data Architecture for Module 1 is complete").
