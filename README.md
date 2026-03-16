# Agentic RAG Email Processor: Vendor Compliance Audit
## Business Scenario
In corporate procurement, verifying that vendor documents (like Compliance Policies or Service Agreements) align with internal company standards is a high-latency manual task.

The Solution: This system automates the "First-Pass Audit." It ingests a company's internal Vendor Compliance Policy, listens for incoming vendor emails, extracts key terms from their attachments, and uses Retrieval-Augmented Generation (RAG) to cross-reference those terms against internal requirements.
It doesn't just extract data; it validates it against a knowledge base to determine if a vendor is "Compliant", "Non-Compliant" or requires "Human Review."

## System Architecture
The system uses a decoupled dual-pipeline architecture to ensure reliability and data privacy:

Ingestion Pipeline (Ingestion Workflow.json):

Parses the vendor_compliance_policy_documents.pdf.

Handles text chunking and metadata tagging.

Populates the local knowledge base (Source of Truth).

Agentic Audit Pipeline (Generation Workflow.json):

Trigger: Webhook/Email intake with file attachment.

Extraction: LLM-powered extraction of structured JSON (Entity names, dates, specific clauses).

RAG Enrichment: Queries the knowledge base for specific internal rules relevant to the extracted entities.

Decision Logic: Validates extracted terms against retrieved rules. The agreement may be accepted automatically, rejected or flagged as "human intervention"

Routing: High-confidence matches are stored in google spreadsheets in different sheets according to the extracted information; ambiguities or compliance failures are flagged for human intervention.

    graph TD
    
    %% Trigger and Intake
    A[Email Attachment Received] --> B{Filter Node}
    
    B -- No PDF --> C[Notify Sender: Missing File]
    B -- PDF Found --> D[OCR: Extract Document Text]

    %% The Agentic Loop
    D --> E[Agent: Compliance Auditor]
    
    subgraph "Agentic RAG Core"
    E <--> F[(Knowledge Base: Internal Policies)]
    end

    %% Decision and Routing
    E --> G[Sanitizer: JSON Formatter]
    G --> H{Router: Triage Results}

    H -- "COMPLIANT" --> I[Append to 'Compliant' Sheet]
    H -- "NON_COMPLIANT" --> J[Append to 'Rejection' Sheet]
    H -- "HUMAN_REVIEW" --> K[Alert: Manual Review Required]

    %% Final Reporting
    I --> L[Report: Final Audit Summary]
    J --> M[Notify: Automated Rejection Email]
    K --> L
    M --> L
## Optimization & Robustness
To move beyond a basic prototype, I implemented several production-grade optimizations:

1. Cost & API Optimization (Batch Processing)
To minimize token usage and API costs:

Attachment Validation: The system first checks for the presence and file type of attachments. If a PDF is missing, the process terminates immediately before calling the LLM, saving unnecessary costs.

Consolidated Extraction: For emails containing multiple documents, the system batches text extraction into a single context window where possible, reducing the number of individual API calls to Gemini.

2. Exception Handling: Missing or Corrupt Files
Input Guardrails: If an email is received without a PDF attachment, the system triggers a "Missing Document" log.

Graceful Degradation: Instead of failing silently, the system identifies the sender and flags the record in the spreadsheet as "Invalid Submission," preventing the agent from "hallucinating" data from an empty input.

3. Finalized Notification & Logging
End-of-Process Execution: To prevent spam and ensure data integrity, email notifications are sent only at the very end of the workflow.

System Logging via Mail: Every execution (Success, Failure, or Human Review required) triggers a structured email notification. This acts as a real-time audit log for administrators to monitor the health of the agent.

## Limitation and future roadmap
This system is designed as a robust, production-ready MVP. To transition this architecture into an enterprise-grade service for external clients, the following enhancements are prioritized:
1. Concurrency & Data Integrity
Current State: The system processes incoming webhooks in real-time. Under high-volume bursts, this can lead to race conditions when writing to external sinks (e.g., Google Sheets API).

Future: Implementing a Message Queue (Redis/RabbitMQ) to decouple ingestion from processing. This ensures "Exactly-Once" delivery and atomic writes, allowing the system to scale horizontally without data duplication.

2. Advanced Retrieval (Reranking)
Current State: Uses standard vector-based similarity search.

Future: Integrating a Cross-Encoder Reranker (e.g., Cohere) to refine the top-k results. This is critical for legal and compliance documents where the semantic nuance of a specific clause can significantly alter the audit outcome.

4. Specialized Model Fine-Tuning
Current State: Relies on Gemini's general-purpose reasoning for extraction.

Future: As the dataset grows, I would transition to a Fine-tuned Small Language Model (SLM). This would provide higher accuracy for niche compliance terminology while reducing latency and operational costs compared to frontier models.

5. Observability & Evaluation Pipelines
Current State: System health is monitored via automated email audit logs.

Future: Integration with LangSmith or Arize Phoenix to track "Golden Dataset" performance, monitor for model drift, and conduct granular error analysis on RAG retrieval accuracy.

## Tech Stack
Orchestration: n8n (Self-hosted via Docker)

LLM Engine: Google Gemini 1.5 Flash

RAG Logic: Vector-based retrieval (Internal n8n Vector Store) with Google Gemini Embeddings

Storage: Local File System & JSON Structured Output

Infrastructure: Docker Compose

## Running the System
1. Prerequisites
Docker Desktop installed.
Google Gemini API Key.
A Google Service Account (for Spreadsheet access).

2. Deployment
Clone the repo:
git clone https://github.com/Pilmesha/N8N-Automation
cd N8N-Automation
Setup Environment:
Create a .env file

3. Launch:
docker-compose up -d

4. Import:
Navigate to http://localhost:5678
Import the two .json files from /workflows.

5. Initialization:
Run the Ingestion Workflow first to populate the vector store with the documents in /data before sending test emails.
