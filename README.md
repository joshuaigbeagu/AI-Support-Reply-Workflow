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

### Ticket Storage: Airtable Over a Full CRM

Ticket logging initially used HubSpot's contact and ticket objects. It was moved to a single Airtable table because the requirement was to log and track support requests, not manage a sales pipeline.

HubSpot's contact-upsert logic and CRM lifecycle fields introduced complexity that was not necessary for this use case.

Sender email is stored as a plain, filterable field rather than a separate linked Senders table. This is simpler to maintain and is sufficient for queries such as determining how many times a sender has submitted a complaint.

### One Record Per Message

Each processed message creates a new Airtable record rather than updating an existing record for the entire thread.

This avoids search-then-branch-or-create logic and keeps each support interaction independently traceable. Full conversation history remains accessible by filtering records using the Thread ID.

Duplicate-write protection against a rare trigger re-fire was deliberately omitted because adding a search step before every write would introduce additional complexity that was not justified at the current scale.

### Reliable Ticket Logging

Both high-confidence and low-confidence paths converge on ticket logging in Airtable. This ensures that every processed support request is recorded regardless of its outcome.

### Failure-Safe Email Handling

The workflow marks an email as read only after successful ticket logging. If a downstream step fails, the message remains unread and can be retried rather than being silently removed from the processing queue.
