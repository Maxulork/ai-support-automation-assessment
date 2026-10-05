# PulseFit AI Support Automation

## Overview

This repository contains an n8n workflow built for the **AI Support Automation** technical assessment.

The workflow processes a batch of synthetic PulseFit support tickets, validates each ticket, normalizes customer data, classifies the request with an LLM, applies deterministic branch-specific business logic, generates a customer-facing response, validates the output, and logs the result.

The implementation is intentionally structured so that:

- **AI is used for natural-language understanding and response generation.**
- **Deterministic workflow logic is used for validation, subscription matching, eligibility checks, routing, and action decisions.**

No real Zendesk, Stripe, PayPal, or production billing APIs are called. Cancellation and refund actions are simulated.

---

## Repository Contents

```text
.
├── README.md
├── workflow.json
└── sample-input.json
```

Optional screenshots can be added under:

```text
screenshots/
```

---

## Workflow Architecture

High-level flow:

```text
Manual Trigger
    ↓
Load Sample Tickets
    ↓
Validate Ticket
    ↓
Is Ticket Valid?
   ├── Invalid → Build Invalid Ticket Result
   │
   └── Valid
        ↓
   Normalize Email
        ↓
   AI Classifier
        ↓
   Merge Ticket + Classification
        ↓
   Extract Classification
        ↓
   Route by Intent
        ├── Cancellation
        ├── Cancellation + Billing Clarification
        ├── Refund
        ├── Technical Issue
        └── Other
        ↓
   Merge Branch Results
        ↓
   AI Response Generator
        ↓
   Merge Result + Customer Response
        ↓
   Extract / Validate Customer Response
        ↓
   Build Final Valid Result

Invalid Result + Valid Results
        ↓
   Merge Final Results
        ↓
   Prepare Log Entry
        ↓
   Data Table Logging
```

---

## How to Run

### Requirements

- n8n
- Access to an OpenAI or compatible LLM credential
- n8n Data Table support

### Steps

1. Import `workflow.json` into n8n.
2. Configure the LLM credential used by:
   - `AI Classifier`
   - `AI Response Generator`
3. Create or select the logging Data Table used by the workflow.
4. Use the provided `sample-input.json` as the reference input dataset.
5. Run the workflow from the `Manual Trigger`.

For the assessment version, the sample ticket batch is loaded through a test input node for reproducible execution.

In production, the workflow would receive tickets from Zendesk or another support platform instead of loading the embedded sample batch.

---

## Input Validation

Each ticket is processed independently.

The following fields are required:

- `ticket_id`
- `message`
- `product`

If a required field is missing, null, or empty:

- the ticket is marked as failed,
- a useful `error_message` is created,
- the failed ticket is still logged,
- the remaining tickets continue processing.

This is demonstrated by sample ticket `10005`, which has a null `message`.

---

## Email Normalization

Customer email normalization is handled deterministically before AI processing.

The workflow:

1. trims surrounding whitespace,
2. converts the email to lowercase,
3. removes Gmail `+alias` suffixes.

Example:

```text
Anna.Test+trial@gmail.com
→
anna.test@gmail.com
```

No AI is used for email normalization.

---

## AI Classifier

The first LLM call classifies each valid ticket into exactly one intent:

- `cancellation`
- `cancellation_with_billing_clarification`
- `refund`
- `technical_issue`
- `other`

The classifier returns structured JSON:

```json
{
  "intent": "refund",
  "confidence": 0.90
}
```

The AI classifier does **not**:

- count subscriptions,
- determine refund eligibility,
- choose a subscription,
- decide whether cancellation can be executed.

Those decisions are handled deterministically after classification.

---

## Deterministic Business Logic

### Subscription Matching Rule

`matching_subscription_count` includes **all subscriptions whose product matches the ticket product**, regardless of subscription status.

This calculation is never delegated to AI.

---

## Cancellation

The workflow:

1. counts all matching subscriptions,
2. filters matching active subscriptions,
3. applies the following rules:

```text
Exactly 1 active match  → simulated_success
0 active matches        → not_executed
More than 1 active      → requires_review
```

When exactly one active subscription exists, the subscription ID is preserved in the branch result.

---

## Cancellation With Billing Clarification

This branch applies the same cancellation validation rules as the cancellation branch.

It also preserves:

- `billing_provider`
- billing-related context for the customer-response LLM

Example from the provided sample data:

```text
ticket_id: 10002
billing_provider: paypal
```

---

## Refund

For this assessment, a refund is eligible when at least one subscription matches the ticket product.

Rules:

```text
At least 1 matching subscription → simulated_success
No matching subscriptions        → not_executed
```

Subscription status is not used for refund eligibility in this assessment.

Example:

Ticket `10003` contains:

- one active `pulsefit` subscription,
- one inactive `pulsefit` subscription,
- one active `otherapp` subscription.

Therefore:

```text
matching_subscription_count = 2
```

---

## Technical Issues

Technical knowledge-base matching is deterministic.

Mock knowledge base:

```text
freezing
→ Ask the customer to update the app, restart the device, and retry.

login
→ Ask the customer to verify the account email and reset the password.
```

If a matching topic is found:

```text
action_type = technical_guidance
```

If no knowledge-base entry is found:

```text
action_type = manual_review
action_status = requires_review
```

---

## Other Requests

Requests classified as `other` do not trigger billing or subscription actions.

They are routed to general support:

```text
action_type = manual_review
action_status = queued
```

---

## Standardized Branch Result

Before customer-response generation, all branches return a consistent structure containing fields such as:

```json
{
  "matching_subscription_count": 1,
  "action_type": "cancellation",
  "action_status": "simulated_success",
  "details": {
    "reason": null
  }
}
```

Possible action statuses include:

- `simulated_success`
- `not_executed`
- `requires_review`
- `queued`

---

## AI Customer Response Generator

The second LLM call receives:

- original ticket subject,
- original customer message,
- detected intent,
- matching subscription count,
- action type,
- action status,
- branch-specific details.

It returns structured JSON:

```json
{
  "subject": "Your support request",
  "message": "..."
}
```

The prompt explicitly instructs the model to:

- avoid exposing internal workflow details,
- avoid claiming success unless the workflow result indicates success,
- avoid inventing billing or subscription information,
- avoid inventing response-time promises,
- avoid adding unsupported next steps,
- reflect `queued`, `not_executed`, and `requires_review` states accurately.

The workflow then extracts and validates the structured customer response before assembling the final ticket result.

---

## Final Result Structure

Successfully processed tickets contain enough information to understand:

- ticket ID,
- normalized email,
- detected intent,
- classifier confidence,
- matching subscription count,
- action type,
- action status,
- branch details,
- generated customer response,
- execution status,
- error message.

Example:

```json
{
  "ticket_id": 10006,
  "normalized_customer_email": "alex@example.com",
  "detected_intent": "other",
  "confidence": 0.9,
  "matching_subscription_count": 0,
  "action_type": "manual_review",
  "action_status": "queued",
  "details": {
    "reason": "General support request requires manual review"
  },
  "customer_response": {
    "subject": "Re: Question about workouts",
    "message": "Thanks for reaching out. Your question has been forwarded to our support team for manual review."
  },
  "execution_status": "Success",
  "error_message": null
}
```

Invalid tickets use the same general result model where possible and include a useful error message.

---

## Logging

Every processed ticket is written to an n8n Data Table.

The log contains at least:

- `timestamp`
- `ticket_id`
- `detected_intent`
- `matching_subscription_count`
- `action_type`
- `action_status`
- `execution_status`
- `error_message`

Example failed ticket:

```json
{
  "ticket_id": 10005,
  "detected_intent": null,
  "matching_subscription_count": 0,
  "action_type": "none",
  "action_status": "not_executed",
  "execution_status": "Failed",
  "error_message": "Missing or empty required fields: message"
}
```

A failed ticket does not stop processing of the rest of the batch.

---

## AI vs Deterministic Logic

### AI is used for

1. Intent classification
2. Customer-facing response generation

### Deterministic workflow logic is used for

- required-field validation,
- email normalization,
- subscription matching,
- active subscription checks,
- refund eligibility,
- technical knowledge-base matching,
- branch routing,
- action status decisions,
- final result assembly,
- logging.

This separation prevents the LLM from making business-critical billing decisions.

---

## Low-Confidence Classifier Handling

Confidence-based routing is not required for this assessment and is not used to block the provided sample tickets.

In production, low-confidence results should not automatically trigger high-impact actions.

A production approach could be:

```text
confidence >= 0.80
→ continue to deterministic processing

confidence < 0.80
→ route to manual review
```

Additional safeguards could include:

- a second classification pass,
- deterministic keyword/rule checks,
- human approval before billing actions,
- a dedicated review queue.

---

## Error Handling

The workflow is designed so that one invalid ticket does not stop the full batch.

Current handling includes:

- ticket-level required-field validation,
- explicit failed-ticket results,
- structured error messages,
- continued processing of valid tickets,
- persistent logging of failed and successful tickets.

For production, I would additionally add:

- retries for transient LLM/API failures,
- exponential backoff,
- a dedicated error workflow,
- dead-letter handling for permanently failed executions.

---

## Zendesk Integration

In production, Zendesk could trigger the workflow using a Zendesk webhook or an n8n Webhook endpoint.

A typical production flow would be:

```text
Zendesk Trigger / Webhook
→ n8n Webhook
→ retrieve ticket/customer context if required
→ process ticket
→ generate response
→ write internal note or customer reply back to Zendesk
```

If additional data is required, the workflow could call the Zendesk API using an authenticated HTTP Request node or a Zendesk integration.

### Credential Management

API credentials should never be hard-coded in workflow code or committed to GitHub.

Credentials for services such as:

- Zendesk
- OpenAI / LLM provider
- billing backend
- Stripe / PayPal

should be stored using n8n Credentials or another secret-management solution.

Production credentials should use:

- least-privilege permissions,
- separate credentials per service,
- restricted scopes,
- regular rotation.

---

## Three Most Important Production Improvements

Before deploying this workflow for approximately 10,000 customer tickets per month, especially if it can eventually trigger real cancellations or refunds, I would prioritize the following three improvements.

### 1. Human Approval and Risk Controls

High-impact actions such as refunds and cancellations should have additional safeguards.

Examples:

- low-confidence classifications → manual review,
- multiple matching subscriptions → manual review,
- unusual or high-value refunds → manual approval,
- ambiguous billing states → manual review.

The LLM should never directly control money-changing operations.

---

### 2. Idempotency, Retry Logic, and Failure Recovery

Production billing actions must be protected against duplicate execution.

I would add:

- idempotency keys,
- controlled retries,
- exponential backoff,
- timeout handling,
- dedicated error workflows,
- dead-letter handling.

This prevents situations such as a workflow retry causing the same refund or cancellation to happen twice.

---

### 3. Monitoring, Auditability, and Security

Production deployment should provide full observability.

I would add:

- structured centralized logs,
- correlation/execution IDs,
- operational metrics,
- alerting,
- audit history for every billing action,
- credential rotation,
- least-privilege API access,
- clear tracing from Zendesk ticket → workflow execution → backend action.

For billing-related automation, every action should be explainable and auditable.

---

## Known Limitations

- Billing and subscription actions are simulated.
- The technical knowledge base contains only the assessment's mock entries.
- No real Zendesk integration is included.
- No real Stripe or PayPal API calls are made.
- Confidence-based routing is described but not implemented.
- The workflow is optimized for the assessment dataset rather than production-scale traffic.
- The sample batch is loaded through a manual test path for reproducible assessment execution.

---

## Sample Dataset Coverage

The provided sample dataset exercises all major paths:

```text
10001 → cancellation
10002 → cancellation_with_billing_clarification
10003 → refund
10004 → technical_issue
10005 → invalid ticket / failed validation
10006 → other / manual review
```

This makes it possible to verify all main workflow branches in a single execution.
