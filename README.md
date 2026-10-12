# AI Sales CRM Automation

An n8n workflow that captures incoming leads, qualifies them with Gemini, organizes them in HubSpot, and routes follow-up actions based on lead intent. High-intent leads should not get lost because someone forgot to check an inbox or follow up on time.

**Demo:** https://lnkd.in/p/dFSMQQ2y

## What it does

The workflow takes a new lead from an intake form and automatically:

- normalizes the submitted lead data
- analyzes the lead message with Gemini
- classifies the lead as Hot, Warm, Cold, or Needs Review
- scores the lead and extracts sales context
- creates or updates the contact in the CRM
- finds or creates the related company and associates it with the contact
- routes the lead into the correct sales path
- creates a qualified deal for Hot leads
- drafts a personalized reply
- creates follow-up tasks and sends internal alerts
- checks whether important leads were followed up, and escalates missed Hot leads
- nurtures Cold leads
- sends uncertain leads to a human for review
- sends failure alerts through a separate error workflow

## Lead qualification

Gemini returns structured data for every lead:

- Classification
- Score
- Service interest
- Urgency
- Budget signal
- Purchase intent
- Qualification reason

```json
{
  "classification": "Hot",
  "score": 95,
  "serviceInterest": "Website Redesign",
  "urgency": "High",
  "budgetSignal": "Positive",
  "purchaseIntent": "Strong",
  "reasoning": "The lead has a clear project, deadline, pricing request, and intent to begin soon."
}
```

## Routing logic

**Hot leads**
- contact and company stored in the CRM
- qualified deal created, contact lifecycle moved to Opportunity
- AI qualification data stored on the deal
- Gemini drafts a suggested reply
- urgent follow-up task created and internal alert sent
- workflow waits for the response SLA, then checks the CRM task again
- if the task is still open, an escalation alert is sent

**Warm leads**
- stored as CRM leads with a follow-up task and internal alert
- 24-hour follow-up SLA, with a reminder if the task is still open

**Cold leads**
- stored in the CRM, no deal created
- delayed nurture flow with an AI-generated low-pressure message

**Needs Review**
- used when the AI cannot confidently understand the inquiry
- the lead is not forced into a sales stage
- a manual review task is created, the team is alerted, and a reminder is sent if the review is not done

## CRM lifecycle

New contacts enter the CRM as `Lead`. Only when a Hot lead becomes a qualified deal does the contact move to `Opportunity`, so the lifecycle reflects real sales activity instead of advancing every inquiry.

## Company matching and duplicates

Contacts are created or updated by email address. When a website is provided, the workflow extracts the domain (`https://www.example.com/about` becomes `example.com`), searches the CRM for that company, and either uses the existing record or creates a new one before associating it with the contact.

## Architecture

```text
Lead Intake
    |
Normalize Lead Data
    |
Gemini Qualification
    |
Parse Structured AI Data
    |
Create / Update CRM Contact
    |
Find / Create Company
    |
Associate Contact + Company
    |
Route by Classification
    |
    +----------------+----------------+----------------+
    |                |                |                |
   HOT              WARM             COLD        NEEDS REVIEW
    |                |                |                |
 Create Deal      Follow-up Task     Wait         Review Task
    |                |                |                |
 AI Reply          Alert           AI Nurture        Alert
    |                |                |                |
 Follow-up Task     SLA              Email            SLA
    |                |                                 |
   SLA            Reminder                         Reminder
    |
 Escalation
```

## Response SLAs

| Lead type | Timing |
|---|---|
| Hot | 10 minutes |
| Warm | 24 hours |
| Needs Review | 2 hours |
| Cold nurture | 7 days |

These are configurable per client.

## Reliability

- retry on fail for the Gemini and CRM calls
- structured JSON parsing with a fallback
- uncertain AI responses go to Needs Review instead of being guessed
- a separate n8n Error Trigger workflow emails an admin when the main workflow fails

## Tech stack

- n8n (self-hosted in Docker)
- Gemini API
- HubSpot
- Gmail
- n8n Forms

The same structure can be adapted to other CRMs with an API, such as GoHighLevel, and to other triggers like Calendly.

## Running locally

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

Open `http://localhost:5678`, import the workflow JSON files from the `workflows` folder, and connect your own credentials for Gemini, HubSpot and Gmail.
