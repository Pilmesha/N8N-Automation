# Agentic RAG Email Processor: Vendor Compliance Audit
## Business Scenario
In corporate procurement, verifying that vendor documents (like Compliance Policies or Service Agreements) align with internal company standards is a high-latency manual task.

The Solution: This system automates the "First-Pass Audit." It ingests a company's internal Vendor Compliance Policy, listens for incoming vendor emails, extracts key terms from their attachments, and uses Retrieval-Augmented Generation (RAG) to cross-reference those terms against internal requirements.
It doesn't just extract data; it validates it against a knowledge base to determine if a vendor is "Compliant" or requires "Human Review."

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


## Tech Stack
Orchestration: n8n (Self-hosted via Docker)

LLM Engine: Google Gemini 3.1

RAG Logic: Vector-based retrieval (Internal n8n Vector Store) with Google Gemini Embeddings

Storage: Local File System & JSON Structured Output

Infrastructure: Docker Compose

## Repository Structure
├── data/               # Internal reference PDF for RAG
├── sample_data/        # 4 Test cases (Compliant, Non-Compliant, Messy, Error)
├── workflows/          # Exported n8n JSON files
├── docker-compose.yml  # Container orchestration
└── README.md           # Documentation

## Running the System
1. Prerequisites
Docker Desktop installed.
Google Gemini API Key.
A Google Service Account (for Spreadsheet access).

2. Deployment

2.1. Clone the repo:
git clone https://github.com/Pilmesha/N8N-Automation
cd N8N-Automation

2.2 Setup Environment:
Create a .env file

3. Launch:
docker-compose up -d

4. Import:
Navigate to http://localhost:5678
Import the two .json files from /workflows.
=======
# N8N-Automation
An Agentic RAG system for automated email processing, built with n8n and Docker. Features a dual-pipeline architecture for document ingestion and context-aware response generation.
>>>>>>> d88352de3449709f2a25605797128addb809935e
