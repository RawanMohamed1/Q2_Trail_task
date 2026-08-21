# KYC Rejection Assistant

An n8n Agent + workflow that turns a failed KYC (identity verification) check into:

1. A short **internal note** for the support agent.
2. A friendly, on-brand **customer message** asking them to fix the issue.

A human supporter always reviews (and can edit) the customer-facing message in Slack before
it goes out — the agent never messages the customer directly.

## What's in this repo

| File | What it is |
|---|---|
| `n8n/kyc-rejection-assistant.agent.json` | Reference export of the n8n **Agent** config (system prompt, model, personalisation). Credentials are stripped. |
| `n8n/kyc-rejection-assistant.workflow.json` | Importable n8n **workflow** JSON for the approval flow. Credentials are stripped. |

No API keys, tokens, or credential IDs are included anywhere in this repo. You'll connect your
own during setup below.

## How it works

```
[Form Trigger]  →  [Draft KYC Message]  →  [Get Supporter Approval]  →  [Send to Customer]
 you fill in         calls the "KYC          Slack DM to a supporter      Slack DM with the
 the rejection       Rejection Assistant"    with the draft — they        supporter's final
 reason              Agent, gets back an     approve as-is or edit        (approved/edited)
                      internal note +        the text, then submit        text
                      customer message
```

- **Form Trigger** — a web form (no coding) where you enter the customer's name, the rejection
  reason, and the customer's Slack username.
- **Draft KYC Message** — calls the published *KYC Rejection Assistant* Agent and gets back
  structured `internal_note` + `customer_message` fields.
- **Get Supporter Approval** — DMs a fixed supporter in Slack using n8n's built-in
  "Send and Wait" (custom form). The message is pre-filled and editable; the workflow pauses
  until they submit.
- **Send to Customer** — DMs whatever the supporter submitted (approved or edited) to the
  customer.

## Setup

### 1. Prerequisites

- An n8n instance (n8n Cloud or self-hosted) with the **Agents** feature available.
- An API credential for an LLM provider (this Agent was built for Anthropic/OpenAI-style
  chat models — any provider n8n supports works; the `model` field just needs updating to match).
- A Slack app/bot installed in your workspace with:
  - Bot token scopes: `chat:write`, `im:write`, `users:read`, `users:read.email`
  - **Interactivity & Shortcuts** turned on, with the Request URL pointed at your n8n instance
    (n8n shows you the exact URL once you add the Slack node.

### 2. Add your credentials in n8n

In n8n, go to **Credentials → New** and add:
- Your LLM provider credential (e.g. Anthropic API key).
- A Slack API credential using your bot token.

### 3. Create the Agent

n8n Agents aren't imported from a file — create one from the reference JSON:

1. In n8n, go to **Agents → New Agent**, name it `KYC Rejection Assistant`.
2. Paste the `instructions` field from `n8n/kyc-rejection-assistant.agent.json` into the
   Agent's instructions.
3. Set the model to your chosen provider/model and attach the credential you added in step 2.
4. (Optional) Copy the `personalisation` icon/gradient for a matching look.
5. (Optional) Under **Integrations**, connect Slack with your Slack credential if you want
   people to also be able to chat with the Agent directly in Slack.
6. Validate, then **Publish** the Agent. Note its Agent ID (visible in the URL) — you'll need
   it in step 4.

### 4. Import the workflow

1. In n8n, go to **Workflows → Import from File** and select
   `n8n/kyc-rejection-assistant.workflow.json`.
2. Open the **Draft KYC Message** node and replace `REPLACE_WITH_YOUR_AGENT_ID` with the Agent
   ID from step 3 (use the picker to select "KYC Rejection Assistant" instead of typing it
   by hand).
3. Open **Get Supporter Approval** and **Send to Customer**, and set both nodes' Slack
   credential to the one you added in step 2.
4. In **Get Supporter Approval**, replace `REPLACE_WITH_SUPPORTER_SLACK_USER_ID` with the
   Slack user who should review/approve messages (use the picker to search by name).
5. Save and **activate** the workflow.

### 5. Try it

Open the Form Trigger's production URL (or run the workflow manually from the editor), fill in
a test rejection reason, and check Slack:
- The supporter gets a DM with the draft — they can submit as-is or edit the text first.
- Whatever they submit gets DM'd to the "customer" Slack username you entered.

### Notes / things to adapt for production

- The customer is currently reached via **Slack DM**, using the Slack username entered in the
  form. If your real customers aren't in your Slack workspace, swap the final **Send to
  Customer** node for an email node (e.g. Gmail/SMTP) instead, keyed off a customer email
  field in the form.
- The supporter is currently a single fixed Slack user. For a team, point it at a Slack
  **channel** instead (`select: channel` on the Slack node) so anyone on the team can respond.
