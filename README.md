# AI Support Reply Workflow

An AI-powered customer support workflow that analyzes inbound customer support emails, retrieves relevant information from a knowledge base, drafts replies or escalates to a human based on confidence, and logs every ticket for tracking.

## Overview

Support teams often spend significant time answering repetitive customer questions, searching documentation, drafting responses, and manually logging and tracking support requests.

## Objective

The workflow is designed to reduce repetitive support work while maintaining human oversight for cases where the AI does not have enough information or confidence to provide a reliable response.

The system separates support requests into two paths:

- **High-confidence requests:** Generate a response draft for human review.
- **Low-confidence requests:** Escalate the request to a human support agent for further handling.

Every request is also logged so support activity can be tracked and reviewed.


## Workflow Architecture

```
                                        Inbound Customer Email (unread, inbox)
                                                         ↓
                                    Extract Data (sender, subject, body, thread ID)
                                                         ↓
                   AI Agent (searches knowledge base, evaluates confidence, generates structured output)
                                                         ↓
                                   Confidence Branch (routes on structured output: status)
                                         ↓                                     ↓
                                    High Confidence                     Low Confidence
                                         ↓                                     ↓
                                   Draft Email Reply                   Escalate to Human
                                         ↓                                     ↓
                                         └──────────→ Ticket Logged ←──────────┘
                                                          ↓
                                                 Email Marked as Read
```

## Tools & Technologies

- **n8n** — Workflow orchestration and automation
- **Gmail** — Email trigger, draft creation, and message read-status tracking
- **OpenAI** — AI agent reasoning and structured output generation
- **Pinecone** — Vector database for knowledge base retrieval
- **Slack** — Human escalation alerts for low-confidence requests
- **Airtable** — Ticket logging and support request tracking


## Key Features

- **Automated Email Intake** — Monitors Gmail for unread customer support emails in the inbox.
- **Contextual Knowledge Retrieval** — Retrieves relevant information from the knowledge base before generating a response.
- **AI-Powered Response Generation** — Uses OpenAI to analyze requests and generate structured response output.
- **Confidence-Based Routing** — Classifies each request as high or low confidence based on whether the retrieved knowledge supports a reliable answer, rather than allowing the AI to guess.
- **Human Review** — Creates a Gmail draft for high-confidence requests instead of sending responses automatically.
- **Low-Confidence Escalation** — Sends low-confidence requests to Slack for human intervention.
- **Centralized Ticket Logging** — Records support requests and their outcomes in Airtable.
- **Reliable State Tracking** — Marks messages as read only after successful ticket logging, so failures leave the email unread for retry instead of silently dropping the request.
- **Execution Error Alerts** — Uses a dedicated n8n error workflow to detect failed executions and send alerts for investigation.


## Design Decisions

### Human-in-the-Loop Response Handling

The workflow does not automatically send AI-generated responses to customers. High-confidence requests are converted into Gmail drafts for human review before sending.

This reduces the risk of incorrect or unsupported responses reaching customers while still automating the time-consuming drafting process.

### Knowledge-Grounded Responses

The AI agent retrieves relevant information from the knowledge base before generating a response. A request is only treated as high confidence when the available knowledge provides sufficient support for an answer.

This helps reduce unsupported or hallucinated responses.

### Structured Confidence Routing

The AI agent returns structured output containing a `status` value, validated against a strict JSON schema. The workflow uses this value to determine whether a request should proceed to draft creation or be escalated for human handling.

This makes the routing logic explicit and predictable. The decision is enforced at the data layer rather than relying solely on prompt instructions or free-form AI output.

### Low-Confidence Escalation

Requests that cannot be reliably answered from the available knowledge are escalated through Slack instead of forcing the AI to generate a response.

This provides a clear fallback path for cases requiring human judgment.

### Reliable Ticket Logging

Both high-confidence and low-confidence paths converge on ticket logging in Airtable. This ensures that every processed support request is recorded regardless of its outcome.

### Ticket Storage: Airtable Over a Full CRM

Ticket logging initially used HubSpot's contact and ticket objects. It was moved to a single Airtable table because the requirement was to log and track support requests, not manage a sales pipeline.

HubSpot's contact-upsert logic and CRM lifecycle fields introduced complexity that was not necessary for this use case.

Sender email is stored as a plain, filterable field rather than a separate linked Senders table. This is simpler to maintain and is sufficient for queries such as determining how many times a sender has submitted a complaint.

### One Record Per Message

Each processed message creates a new Airtable record rather than updating an existing record for the entire thread.

This avoids search-then-branch-or-create logic and keeps each support interaction independently traceable. Full conversation history remains accessible by filtering records using the Thread ID.

Duplicate-write protection against a rare trigger re-fire was deliberately omitted because adding a search step before every write would introduce additional complexity that was not justified at the current scale.

### Failure-Safe Email Handling

The workflow marks an email as read only after successful ticket logging. If a downstream step fails, the message remains unread and can be retried rather than being silently removed from the processing queue.

### Execution Error Handling

A dedicated n8n error workflow monitors failed executions and sends alerts when an execution error occurs.

This provides operational visibility into workflow failures without requiring manual inspection of n8n execution history.


## Setup & Configuration

### Prerequisites

- n8n instance
- Gmail account connected to n8n
- OpenAI API access
- Pinecone account and index
- Slack workspace and escalation channel
- Airtable base for ticket logging

### Required Integrations

Before running the workflow, configure the following integrations in n8n:

| Integration | Purpose | Used By |
|---|---|---|
| Gmail | Monitor incoming support emails, create reply drafts, and update message read status | Incoming Email, Create Draft Reply, Mark as Read |
| OpenAI | Generate AI responses and create embeddings for knowledge retrieval | OpenAI Chat Model, OpenAI Embeddings |
| Pinecone | Retrieve relevant information from the support knowledge base | Pinecone Vector Store |
| Slack | Notify human support staff when a request requires escalation | Escalate to Human |
| Airtable | Store and track processed support requests | Create Ticket |

### Knowledge Base

The Pinecone index should contain the support documentation used by the AI agent when answering customer requests.

The knowledge base is structured as focused Q&A content to improve retrieval relevance and provide the AI agent with sufficient context to determine whether a reliable answer is available.

The quality and coverage of the knowledge base directly affect the quality of generated responses and confidence classification.

### Airtable Schema

The `Create Record` node expects a table containing the following fields:

| Field | Type | Description |
|---|---|---|
| Sender Email | Email | Customer email address |
| Sender Name | Single line text | Customer name |
| Subject | Single line text | Original email subject |
| Email Body | Long text | Original customer message |
| Thread ID | Single line text | Gmail conversation/thread identifier |
| Complaint Category | Single select | Support request category |
| Priority | Single select | `Low`, `Medium`, `High`, or `Urgent` |
| AI Status | Single select | `Draft Ready` or `Escalate` |
| Draft Reply | Long text | AI-generated reply draft when applicable |
| Escalation Reason | Long text | Reason provided when human escalation is required |

### Credentials & Secrets

All API credentials and authentication details should be configured through n8n's credential system.

**Do not store API keys, access tokens, passwords, or other secrets directly in the workflow JSON or repository.**


## Testing & Validation

The workflow was tested against support requests designed to validate both the normal response path and the human-escalation path.

### Test Scenarios

| Scenario | Expected Outcome |
|---|---|
| Request clearly answered by the knowledge base | High-confidence classification → Gmail draft created → Ticket logged |
| Request partially supported by the knowledge base | Low-confidence classification → Slack escalation → Ticket logged |
| Request not covered by the knowledge base | Low-confidence classification → Slack escalation → Ticket logged |
| Successful ticket logging | Email marked as read |
| Ticket logging failure | Email remains unread for retry |
| High-confidence request | No automatic email is sent; response remains a Gmail draft |
| Low-confidence request | No customer reply is generated; request is escalated to a human |

### Validation Criteria

The workflow is considered successful when:

- AI responses are grounded in retrieved knowledge rather than unsupported assumptions.
- Structured AI output is accepted only when it matches the expected schema.
- High-confidence requests produce a Gmail draft without automatically sending it.
- Low-confidence requests are escalated through Slack.
- Every processed request is recorded in Airtable.
- Emails are marked as read only after successful downstream processing.
- Failed executions do not silently remove messages from the processing queue.


## Project Structure

```
AI-Support-Reply-Workflow/
├── AI-Support-Reply-Workflow.json
└── README.md
```

## Limitations

- **Human review is still required** — High-confidence responses are drafted but not automatically sent to customers.
- **Knowledge base dependent** — Response quality depends on the accuracy, completeness, and relevance of the information stored in Pinecone.
- **Email-based intake** — The current workflow is designed around Gmail and does not directly process requests from other support channels.
- **Single knowledge source** — The AI currently relies on the configured knowledge base rather than dynamically consulting multiple external sources.
- **Duplicate processing** — The workflow does not currently implement explicit idempotency protection against rare duplicate trigger events.
- **Structured classification is not infallible** — Schema validation ensures the AI output follows the expected structure, but it does not guarantee that the underlying classification is correct.


## Future Improvements

- Add idempotency protection to prevent duplicate ticket creation from repeated trigger events.
- Introduce confidence thresholds based on retrieval quality and response evaluation rather than relying on a single classification status.
- Add automated follow-up handling for unresolved support requests.
- Introduce conversation-level memory for multi-message support threads.
- Add an inbound request classifier to identify whether an email is a genuine support request before sending it through the full AI processing pipeline.
