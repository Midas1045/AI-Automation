## AI Email Assistant: Intelligent Email Management (n8n)

An n8n workflow that watches a Gmail inbox, uses AI to read each incoming email, classifies it, sets a priority, summarizes it, and writes a professional **draft reply for a person to review**. It labels each email in Gmail, logs it to Google Sheets, and escalates to a human on Telegram when there is no draft to approve. **The workflow never sends an email by itself.**

#### Workflow screenshot

<img width="1711" height="567" alt="image" src="https://github.com/user-attachments/assets/fa847b6a-afb1-4e99-ad43-08d5b7e4022e" />


#### 1. Objectives

- Detect incoming emails in Gmail automatically.
- Classify each email as Sales, Support, Billing, Partnerships, Jobs or Others.
- Assign a priority (High, Medium or Low) and write a short summary.
- Flag emails that need human judgement, with a reason.
- Generate a professional HTML reply and save it as a Gmail draft.
- Apply the matching Gmail label to the incoming email.
- Record every processed email in a category-specific Google Sheets tab.
- Escalate to a human on Telegram when no draft reply exists.
- Report failures through a separate error-handling workflow.

#### 2. Technologies

| Technology | Purpose |
|---|---|
| n8n | Workflow automation |
| Gmail | Reading emails, labels, drafts |
| OpenAI (`gpt-5-mini`, through the n8n AI Agent node) | Classification, summary and reply writing |
| Structured Output Parser | Forces a consistent JSON answer from the AI |
| Google Sheets | Email classification log, one tab per category |
| Telegram | Escalation alerts for human review |

#### 3. Workflow architecture

```
Gmail Trigger (polls every minute)
   -> Get message -> Edit Fields (email brief) -> AI Agent (OpenAI + Structured Output Parser)
   -> CATEGORY (Switch, 6 outputs)
         |-> Merge (6 inputs) -> Google Sheets log (tab = category)
         '-> PRIORITY (Switch) -> Add Gmail label -> IF draft reply is not empty
                                                        |-- true  -> Create Gmail draft (HTML)
                                                        '-- false -> Telegram alert (subject, summary, review reason)
```

| Node | Purpose |
|---|---|
| **INBOX RADAR** (Gmail Trigger) | Polls Gmail every minute for new emails. |
| **FULL MESSAGE PULL** (Gmail: Get) | Fetches the full message: sender, subject, body and message ID. |
| **EMAIL BRIEF BUILDER** (Edit Fields) | Creates `FROM`, `SUBJECT`, `BODY` and `MESSAGE` for the AI. |
| **TRIAGE AND REPLY AGENT** (AI Agent) | Classifies the email and drafts the reply. Uses the OpenAI Chat Model and the Structured Output Parser. |
| **CATEGORY** (Switch) | Routes by category into six outputs: SALES, SUPPORT, BILLING, PARTNERSHIPS, JOBS, OTHERS. |
| **Merge** | Brings the six category branches back into one stream so a single Sheets node logs every email. |
| **EMAIL CLASSIFCATION LOG** (Google Sheets) | Appends a row to the tab named after the category. |
| **PRIORITY** (Switch) | Splits by LOW, MEDIUM and HIGH. All outputs continue to labeling. |
| **MESSAGE LABELS** (Gmail) | Adds the category's Gmail label to the incoming email. |
| **If** | Checks whether the AI wrote a draft reply. |
| **REPLY DRAFT BRANCH** (Gmail: Create draft) | Saves the HTML reply as a draft. There is no send operation in the workflow. |
| **Send a text message** (Telegram) | Escalates to a person when there is no draft reply. |

#### 4. AI classification

The agent returns this JSON, enforced by the Structured Output Parser:

```json
{
  "category": "Support",
  "priority": "Medium",
  "subject": "[SUPPORT] Original Subject",
  "summary": "One or two sentences.",
  "draft_reply": "<p>Hi Sam,</p><p>...</p>",
  "needs_review": false,
  "review_reason": ""
}
```

| Field | Description |
|---|---|
| category | Sales, Support, Billing, Partnerships, Jobs or Others |
| priority | High, Medium or Low |
| subject | Subject line for the reply, starting with the category in brackets |
| summary | Short summary of the email |
| draft_reply | The reply in HTML, or empty when no reply is needed |
| needs_review | True when a person must decide |
| review_reason | Why a person must decide (empty when `needs_review` is false) |

#### Human review logic

`needs_review` is set to true, and the draft is left empty, for:

| Email type | Result |
|---|---|
| Job applications and recruitment | Human review |
| Financial disputes or payment discrepancies | Human review |
| Emails needing technical investigation | Human review |
| Partnership or business proposals | Human review |
| Complaints, sensitive matters, decisions needing judgement | Human review |
| Routine enquiries | Draft reply, no review |
| Automated notifications that need no reply | No draft |

The full system prompt is in [`system-prompt.md`](system-prompt.md).

#### 5. Gmail integration

- **Retrieve:** the Get Message node reads sender name and address, subject, body and the Gmail message ID.
- **Labels:** the label node adds the category label to the incoming email, using its Gmail message ID. The Gmail node needs label **IDs** (such as `Label_123...`), not names, and the correct ID is picked with a lookup on the category.
- **Drafts:** the reply is saved as an HTML draft, with the AI-generated subject and the sender as the recipient. A person reviews and sends it manually.

#### 6. Human review and escalation

When the AI leaves `draft_reply` empty (the email needs review, or it is an automated notification), the **If** node sends the item to Telegram. The alert contains the subject, the summary and the review reason.

#### 7. Google Sheets log

One spreadsheet with six tabs: Sales, Support, Billing, Partnerships, Jobs, Others. Every tab has the same columns:

```text
Date Processed | Time Processed | Sender Name | Sender Email | Email Subject | Category | Priority | Email Summary | Needs Review | Review Reason | Gmail Message ID
```

The tab is chosen by an expression, which is why the six branches are merged first:

```text
{{ $json.output.category }}
```

Mappings used:

```javascript
{{ $('FULL MESSAGE PULL').first().json.from.value[0].name }}      // Sender Name
{{ $('FULL MESSAGE PULL').first().json.from.value[0].address }}   // Sender Email
{{ $('FULL MESSAGE PULL').first().json.id }}                      // Gmail Message ID
{{ $now.toFormat('dd LLLL yyyy') }}                               // Date Processed
{{ $now.toFormat('hh:mm') + ' ' + ($now.hour >= 12 ? 'PM' : 'AM') }}   // Time Processed
```

The draft reply isn't stored in the sheet, since Gmail already holds the complete draft. The Gmail API message ID is used, not the email's `Message-ID` header.

#### 8. Error handling

A separate workflow catches failures from this one, logs them to a Google Sheet and sends a Telegram alert. See [`README_Error_Handling.md`](README_Error_Handling.md). It is selected as this workflow's **Error Workflow** in the workflow settings.

#### 9. Testing

I tested the workflow with sample emails. Each one landed where expected: classified into the right category, logged in the right sheet tab, labelled correctly in Gmail, and either saved as a draft reply or escalated on Telegram when no reply existed.

| Test email | Behaviour |
|---|---|
| Enquiry About Bulk Branded Notebooks | Sales, draft reply, label applied |
| Unable to Access My Account | Support, draft reply, label applied |
| Discrepancy in My Latest Invoice | Billing, flagged for review, Telegram escalation |
| Proposal for a Strategic Business Partnership | Partnerships, flagged for review, Telegram escalation |
| Application for Junior Operations Associate Position | Jobs, flagged for review, Telegram escalation |
| Automated notification | Others, no draft, Telegram escalation |

#### 10. Setup

1. Create the six labels in Gmail, then run a Gmail "Label: Get Many" node once to copy their IDs.
2. Create the Google Sheet with the six tabs and the columns above.
3. Import the workflow JSON and connect credentials for Gmail, Google Sheets, OpenAI and Telegram.
4. Replace the placeholders in the JSON:

| Placeholder | Replace with |
|---|---|
| `GMAIL_CREDENTIAL_ID`, `GOOGLE_SHEETS_CREDENTIAL_ID`, `TELEGRAM_CREDENTIAL_ID` | Your saved n8n credentials (re-select them after import) |
| `GOOGLE_SPREADSHEET_ID` | Your spreadsheet's ID |
| `GMAIL_LABEL_SALES`, `GMAIL_LABEL_SUPPORT`, `GMAIL_LABEL_BILLING`, `GMAIL_LABEL_PARTNERSHIPS`, `GMAIL_LABEL_JOBS`, `GMAIL_LABEL_OTHERS` | The label IDs from step 1 |
| `TELEGRAM_CHAT_ID` | The Telegram chat that should receive alerts |
| `N8N_INSTANCE_ID` | Leave as is. n8n replaces it on import |

5. Edit the system prompt with your name and company (see `system-prompt.md`).
6. Switch the workflow to **Active**.

#### 11. Results
<img width="1414" height="206" alt="image" src="https://github.com/user-attachments/assets/e34eb10f-9e85-40aa-9440-12749a82987a" />

<img width="1152" height="916" alt="image" src="https://github.com/user-attachments/assets/62bb6c33-ae68-42e7-bfc4-41ac4e1e5ebf" />


#### 12. What I learned

- Gmail's label operations need label IDs, which differ from the names in the Gmail sidebar.
- After a Gmail node runs, `$json` holds that node's output, so earlier data has to be read by naming the node.
- A Structured Output Parser with enums keeps the AI to valid categories, which makes Switch routing reliable.
- Merging branches lets one Sheets node serve every category.
- Treating email text as data helps protect against prompt injection.
- Saving drafts, not sending, keeps a person in control.

#### 12. Notes

- Never publish credential IDs, label IDs, spreadsheet IDs, chat IDs or API keys.
- The name and company in the test prompt are fictional.


