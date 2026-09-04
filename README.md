# AI Ticket Triage System

## Overview

An n8n workflow template that automates the intake, AI classification, prioritization, human approval, and response of customer support tickets, using Google Gemini, Trello, Gmail, and Google Sheets.

## Use Case

Teams that receive support requests through a simple web form and want an AI first pass at triage — checking scope, classifying the ticket, drafting a response, looking up the customer's plan, and routing the ticket on a Trello board — while still keeping a human in the loop to approve every AI-drafted response before it reaches the customer.

## How It Works

1. A customer submits a ticket through the form trigger.
2. A guardrails step checks the message is on-topic and not a jailbreak/prompt-injection attempt.
3. If in scope, an AI Agent classifies the ticket (category, priority, team), looks up the customer's plan in a Google Sheet, and drafts an HTML response.
4. A Trello card is created on the "new tickets" list, labeled by priority.
5. A support team member receives an email with the full ticket context and the AI-drafted response, and approves or rejects it directly from the email.
6. If approved, the response is emailed to the customer and the card is moved to the "Resolved by AI" list.
7. If rejected, the card is moved to a "Manual Review" list and no email is sent to the customer.
8. If the message is out of scope, the workflow ends with no further action.

## Features

- Guardrails-based scope and jailbreak validation before any AI processing
- AI-driven ticket classification (category, priority, responsible team)
- AI-drafted, ready-to-send HTML customer responses
- Customer plan lookup via a Google Sheets AI tool
- Automatic Trello card creation with priority labeling
- Human-in-the-loop approval via a Gmail "send and wait" step
- Automatic routing of Trello cards based on the approval outcome

## Workflow Architecture

```
New Support Ticket (Form Trigger)
  -> Validate Ticket Scope (Guardrails)
       -> [in scope] Classify and Draft Response (AI Agent)
            (tools: Look Up Customer Record / Google Sheets)
            (output parser: Ticket Classification Output Schema)
            -> Create Ticket Card in Trello
                 -> Request Human Approval (Email)
                      -> Is Response Approved?
                           -> [yes] Send Response to Customer -> Move Card to Resolved by AI
                           -> [no]  Move Card to Manual Review
       -> [out of scope] Out of Scope - No Action
```

`Get Trello Lists` and `Get Trello Labels` are standalone helper nodes (not wired into the main flow). Run them manually after importing to discover the list/label IDs of your own Trello board.

## Requirements

- An active n8n instance (self-hosted or cloud)
- A Google Cloud project with the Gemini API enabled
- A Trello account with a board dedicated to ticket triage
- A Google Sheet with a customer database (at least an `email` column and a plan/tier column)
- A Gmail account used to send and receive approval emails

## Required Credentials

| Node(s) | Credential type |
|---|---|
| Google Gemini Chat Model (Guardrails), Google Gemini Chat Model (AI Agent) | Google Gemini (PaLM) API |
| Get Trello Lists, Get Trello Labels, Create Ticket Card in Trello, Move Card to Resolved by AI, Move Card to Manual Review | Trello API |
| Look Up Customer Record (Google Sheets) | Google Sheets API |
| Request Human Approval (Email), Send Response to Customer | Gmail API |

No credentials are included in this template. Connect your own accounts to each node after importing.

## Installation

1. Import `AI-Ticket-Triage-System.json` into a new, empty n8n workflow.
2. Attach your own credentials to every node listed under **Required Credentials**.
3. Follow the **Configuration Guide** below to replace every placeholder ID.
4. Activate the workflow and open the form trigger's production URL to test.

## Configuration Guide

Before activating the workflow, replace the following placeholders (all are plain string values in the JSON, easiest to edit from the node UI after import):

- **Trello board**: `YOUR_TRELLO_BOARD_ID` (in `Get Trello Lists` and `Get Trello Labels`)
- **Trello "new tickets" list**: `YOUR_TRELLO_LIST_ID_NEW_TICKETS` (in `Create Ticket Card in Trello`)
- **Trello priority labels**: `YOUR_TRELLO_LABEL_ID_CRITICAL`, `YOUR_TRELLO_LABEL_ID_HIGH`, `YOUR_TRELLO_LABEL_ID_MEDIUM`, `YOUR_TRELLO_LABEL_ID_LOW` (in `Create Ticket Card in Trello`)
- **Trello "resolved" / "manual review" lists**: `YOUR_TRELLO_LIST_ID_RESOLVED_BY_AI` (in `Move Card to Resolved by AI`) and `YOUR_TRELLO_LIST_ID_MANUAL_REVIEW` (in `Move Card to Manual Review`)
- **Google Sheet**: replace the placeholder spreadsheet URL/ID and re-select your sheet tab in `Look Up Customer Record (Google Sheets)`
- **Approver inbox**: replace `your-approver@example.com` in `Request Human Approval (Email)` with your support team's address
- **Company name**: replace `[Your Company Name]` inside the AI Agent's system message

Run `Get Trello Lists` and `Get Trello Labels` manually (once credentials are attached) to read the IDs of your own board out of their execution output, instead of guessing them.

## AI Classification

The AI Agent returns a structured JSON object (enforced by the `Ticket Classification Output Schema` node) with these fields: `categoria`, `prioridade`, `equipe`, `resumo`, `impacto`, `tem_solucao`, `resposta_cliente_html`, `cliente_existente`, `plano_cliente`.

Categories: `bug`, `feature_request`, `billing`, `account`, `how_to`, `performance`.
Teams: `engenharia`, `produto`, `financeiro`, `customer_success`.
Priorities: `critica`, `alta`, `media`, `baixa`.

All classification rules, category/team/priority definitions, and response-drafting instructions live in the AI Agent's system message and can be edited directly there.

## Ticket Routing

Trello cards are labeled based on the `prioridade` value returned by the AI Agent, using an inline expression in `Create Ticket Card in Trello`. After approval, the card is moved to the "Resolved by AI" list; after rejection, it is moved to "Manual Review".

## Priority Handling

- `critica`: system down, data loss, affects all users
- `alta`: important feature broken, affects the customer's operation — also the minimum priority forced for tickets mentioning "urgent", "production", or "all users", and the level Enterprise/Corporate customers are raised to
- `media`: minor bug, operational question
- `baixa`: suggestion, compliment, simple question

## Testing

1. Submit a test ticket through the form trigger.
2. Confirm the guardrails step passes an in-scope message and blocks an off-topic one.
3. Check that a Trello card appears on your "new tickets" list with the expected priority label.
4. Approve or reject the request from the approval email and confirm the card moves to the correct list.
5. On approval, confirm the customer receives the AI-drafted response by email.

## Security

- The guardrails node screens for jailbreak/prompt-injection attempts and off-topic requests before the ticket reaches the AI Agent.
- No AI-drafted response reaches the customer without explicit human approval.
- This template ships with no credentials, API keys, or personal data. You are responsible for securing the credentials you attach after import.

## Limitations

- The AI Agent can only look up customer data already present in the connected Google Sheet; it does not create or update customer records.
- Approval is binary (approve/reject) — there is no in-place editing of the AI-drafted response before sending.
- The workflow does not implement retries, rate limiting, or SLA tracking.

## Troubleshooting

- **Trello card not created / labels missing**: confirm the placeholder board, list, and label IDs have been replaced with real IDs from your board.
- **Customer lookup returns nothing**: confirm the Google Sheet has an `email` column matching the ticket's email address exactly.
- **Approval email never arrives**: confirm the Gmail credential and the `sendTo` address in `Request Human Approval (Email)` are correctly configured.
- **AI Agent errors on parsing**: confirm the Gemini credential is valid and the `Ticket Classification Output Schema` node has not been modified in a way that breaks the expected JSON shape.

## Customization

- Change classification categories, teams, priorities, or response style by editing the AI Agent's system message.
- Force a single fixed reply language by editing the last line of the system message (see **Configuration Guide**).
- Swap Trello for another task board, or Gmail for another mailbox, by replacing the corresponding nodes and updating the downstream expressions that reference them.

## Repository Structure

```
AI-Ticket-Triage-System.json   Sanitized n8n workflow template (import this file)
README.md                      This file
```

## License

Licensing terms have not yet been defined for this repository.

## Disclaimer

This template is provided as-is for use as a starting point in your own n8n instance. It has not been reviewed for compliance with any specific regulatory framework (e.g. GDPR, LGPD). Review and adapt the data handling, retention, and approval steps to your own requirements before using it in production.

> **Note on screenshots**: no screenshots are included in this repository yet. The previous set was captured from the original private deployment and contained personal data and outdated (pre-sanitization) node names, so it was removed. Fresh screenshots from a clean import of this template can be added here later.
