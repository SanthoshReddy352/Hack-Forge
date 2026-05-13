# **HackForge AI: AI-Accelerated Engineering Playbook**

Building a distributed, event-driven, multi-tenant system like HackForge AI using an AI agent (like Claude 3.5 Sonnet or Gemini 1.5 Pro) requires a strategic approach. If you treat the AI like a magical "build the app" button, the project will collapse under context window degradation, race conditions, and architectural spaghetti.

This document outlines the **AI-Accelerated Engineering** workflow. You are the **Chief Architect and Integrator**; the AI is your **High-Speed Implementer**.

## **1\. Core Rules of Engagement**

Before starting any phase, you must adopt these fundamental rules for working with AI coding agents:

1. **Context Compression, Not Context Dumping:** Do not paste the entire PRD and Architecture into every prompt. Feed the AI *only* the specific contract or document required for the immediate task.  
2. **Session Isolation:** Treat AI chat sessions like database transactions. Start a new chat for a new feature. Long, marathon sessions lead to "context degradation" where the AI forgets earlier constraints and starts hallucinating.  
3. **Contract-First Development:** Never ask the AI to "write the backend." Ask it to "write the OpenAPI YAML." Once approved, ask it to "generate Go structs from this YAML."  
4. **Enforce the CLAUDE.md / System Prompt:** Create a strict rules file in your repository root that defines your tech stack (Go 1.21, Python 3.11, Kafka, PostgreSQL) and strict invariants (e.g., "All random number generation MUST use the seeded wrapper, never the standard library random").

## **2\. Phase-by-Phase Execution Plan**

This aligns directly with the HackForge *Implementation Guide* and *Engineering Roadmap*.

### **Phase 0: Workspace & Contracts (The Foundation)**

**Goal:** Establish the OpenAPI schema, Protobuf definitions, and Kafka Event schemas.

* **Your Job (Human):** \* Initialize the repository structure (hackforge-contracts/, etc.).  
  * Feed the AI the *Microservices API Contract* and *Protobuf Definitions* documents.  
  * Review the generated YAML and .proto files line-by-line for correct types and missing fields.  
* **Claude's Job (AI):** \* Draft the openapi/hackforge-v1.yaml file.  
  * Draft the experiment.proto and scheduler.proto files.  
  * Generate the Avro/JSON schemas for Kafka events.  
* **🚨 Danger Zone (Do Not Trust):** Do not let the AI invent new API routes or change the schema structure. It will try to "helpfully" add CRUD endpoints (like PUT /experiments) that violate your immutable architecture rules.

### **Phase 1: Deterministic Math Core (Python Compute Plane)**

**Goal:** Build the isolated Python statistical engine (NumPy/SciPy).

* **Your Job (Human):** \* Provide the *Mathematical Algorithm Definitions* document.  
  * Instruct the AI to use Test-Driven Development (TDD). Ask for the pytest files *before* the implementation.  
  * Run the tests locally and feed errors back to the AI.  
* **Claude's Job (AI):** \* Write the Kolmogorov-Smirnov (KS) test logic.  
  * Implement the distribution generators (Normal, Poisson, Pareto).  
  * Write the pure Python Causal DAG builder.  
* **🚨 Danger Zone (Do Not Trust):** **The Determinism Trap.** AI models *love* using uuid.uuid4(), time.time(), or the standard random library. You must explicitly audit every Python file to ensure it strictly uses the seeded RNG derived from config\_hash \+ seed. If you miss this, HackForge's core value proposition fails.

### **Phase 2: Control Plane Foundation (Go & PostgreSQL)**

**Goal:** Build the REST API, Database migrations, and Go microservices.

* **Your Job (Human):** \* Provide the *DB Schema & Indexing* and *Multi-Tenant Isolation* documents.  
  * Set up the local docker-compose.yml (Postgres, Redis).  
  * Test the DB migrations locally.  
* **Claude's Job (AI):** \* Write the PostgreSQL DDL migrations (001\_init.sql).  
  * Generate the Go Gin/Fiber API routes mapping to the OpenAPI spec.  
  * Generate the Go gRPC client stubs to communicate with the compute plane.  
* **🚨 Danger Zone (Do Not Trust):** **Row-Level Security (RLS) and Partitioning.** AI often hallucinates PostgreSQL partition syntax or writes insecure RLS policies. You must manually verify that every SQL query includes tenant\_id and that the RLS current\_setting logic is flawless.

### **Phase 3: The Event Backbone (Kafka Integration)**

**Goal:** Connect the Go Control Plane and Python Compute Plane via Kafka.

* **Your Job (Human):** \* Add Kafka to the docker-compose.yml.  
  * Manage the terminal: Monitor Kafka logs to ensure topics are created and messages are flowing.  
* **Claude's Job (AI):** \* Write the Go Kafka Producer (e.g., emitting experiment.created).  
  * Write the Python Kafka Consumer (listening for compute tasks).  
* **🚨 Danger Zone (Do Not Trust):** **Race Conditions and Network Ports.** Claude cannot see your local Docker network. If the Python container cannot resolve the kafka:9092 hostname, Claude might suggest changing the code rather than fixing the Docker network bridge. Always verify infrastructure logic yourself.

### **Phase 4: The Canvas Interface (React/TypeScript)**

**Goal:** Build the interactive frontend.

* **Your Job (Human):** \* Provide the *Designing HackForge AI User Flow* document.  
  * Dictate the state management strategy (e.g., Zustand or Redux).  
* **Claude's Job (AI):** \* Generate Tailwind CSS components for the Dashboard and Dataset Builder.  
  * Implement reactflow for the visual Causal DAG builder.  
  * Write the WebSocket listener for real-time generation progress.  
* **🚨 Danger Zone (Do Not Trust):** AI struggles with complex, multi-step asynchronous state in React. It might create infinite re-render loops when updating the visual graph based on WebSocket events. Review useEffect dependencies rigorously.

## **3\. The "Do Not Trust" Checklist (Architect's Guardrails)**

When reviewing code generated by Claude or Gemini, keep this checklist on your desk:

* \[ \] **The Sycophancy Check:** AI will often agree with you even if you suggest a terrible architectural idea. If you ask it to bypass Kafka and make a direct HTTP call from Go to Python, it will do it, breaking your architecture. **Always be an architect.**  
* \[ \] **The "Silent Rewrite" Check:** Sometimes, when fixing a small bug, the AI will silently rewrite a working function nearby. Always review the full file diff, not just the lines that were broken.  
* \[ \] **The Monolith Temptation:** AI finds it easier to put everything in main.go or app.py. Force it to respect the directory structures outlined in your File\_Structure.md document.  
* \[ \] **The Library Hallucination:** Check go.mod and requirements.txt. AI occasionally imports packages that don't exist, are deprecated, or introduce massive security vulnerabilities.  
* \[ \] **Error Handling Dilution:** AI often writes if err \!= nil { log.Fatal(err) } or except Exception as e: pass. Demand that it writes proper, tenant-aware, structured error logging.

## **Summary**

Building HackForge AI with an AI agent is highly feasible and will easily 10x your development speed. However, **you must own the architecture**. Use the AI to generate the boilerplate, the statistical algorithms, and the UI components, but *you* must stitch the distributed network together, enforce the mathematical determinism, and govern the tenant isolation.