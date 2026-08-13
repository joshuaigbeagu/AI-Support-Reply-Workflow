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
