## Error Handling Workflow (n8n)

A companion n8n workflow that catches failures in the AI Email Assistant, **logs each failure to Google Sheets**, and **sends a Telegram alert** with a link to the failed execution.

#### Error workflow screenshot

<img width="1191" height="326" alt="image" src="https://github.com/user-attachments/assets/ea630d4a-16f7-40e3-b976-6ed3a9369b56" />


#### How it works

```
Error Trigger -> Error Field Mapping (Edit Fields) -> Error Logs (Google Sheets) -> Notify Human Agent (Telegram)
```

| Node | Purpose |
|---|---|
| **ERROR TRIGGER** | Starts whenever a linked workflow fails during an automatic run. |
| **ERROR FIELD MAPPING SCHEMA** (Edit Fields) | Collects the error details and adds the date (`dd/MM/yyyy`) and time (`hh:mm:ss a`). |
| **ERROR LOGS** (Google Sheets) | Appends one row per failure. |
| **NOTIFY HUMAN AGENT** (Telegram) | Sends a formatted alert to a person. |

#### What is recorded

| Column | Source |
|---|---|
| date, time | When the error was handled |
| work_flow name | The name of the failed workflow |
| workflow_id | Its ID |
| failed_node | The last node that ran (`execution.lastNodeExecuted`) |
| error_message | The error text |
| execution_id | The failed run's ID |
| execution_url | A direct link to the failed run |
| execution_mode | How the run was started |
| retry_of | The earlier run this one retried, if any |

#### The Telegram alert

The message contains the workflow name and ID, the failed node, the execution mode, the date and time, the error message, the execution ID, the retry reference and a link to the failed execution, followed by an action line asking someone to investigate.

#### Setup

1. Create a Google Sheet (for example **ERROR LOGS**) with the column headings above in row 1.
2. Import the workflow JSON and connect credentials for Google Sheets and Telegram.
3. Replace the placeholders in the JSON:

| Placeholder | Replace with |
|---|---|
| `GOOGLE_SHEETS_CREDENTIAL_ID`, `TELEGRAM_CREDENTIAL_ID` | Your saved credentials (re-select them after import) |
| `TELEGRAM_CHAT_ID` | The chat that should receive alerts |
| The spreadsheet in the Google Sheets node | Select your own spreadsheet |
| `YOUR_INSTANCE_ID` | Leave as is. n8n replaces it on import |

4. **Link it to the main workflow:** open the AI Email Assistant, go to **Settings**, and set **Error Workflow** to this workflow.
5. Save and activate both workflows. The Error Trigger fires for failures in automatic runs of an active workflow.

#### Results
<img width="492" height="699" alt="image" src="https://github.com/user-attachments/assets/bbd84024-9a6b-4bf9-b9f7-5431ca44fd0e" />
<img width="1813" height="416" alt="image" src="https://github.com/user-attachments/assets/10ad9d71-7b89-4fad-814a-8c867e380147" />


#### Notes

- Never publish credential IDs, chat IDs or spreadsheet IDs. Keep placeholders in shared exports.


