## Daily Motivational Quote Emailer (n8n + Gemini + Gmail)

An automated n8n workflow that generates an original motivational quote with AI and emails it to a list of recipients every day at 8:00 AM and 8:00 PM (Africa/Lagos time).

The morning quote is energizing and action-focused. The evening quote is calm and reflective.

Workflow screenshot

### How it works
Schedule Trigger -> AI Agent (Gemini) -> Get row(s) in Google Sheets -> Gmail
      1 item            1 item                    3 items                 3 emails

Schedule Trigger fires twice a day, at 08:00 and 20:00. AI Agent uses the Google Gemini Chat Model to write one original quote (25 words or fewer). The prompt checks the time of day with $now.hour and switches between a morning and an evening style. Get row(s) in sheet reads the recipient list from Google Sheets. One sheet row becomes one item, so one quote fans out to every recipient. Send a message (Gmail) sends one HTML email per recipient, using the quote written by the agent.

### Tools used
1. n8n for workflow automation
2. Google Gemini (via the AI Agent node) for quote generation
3. Google Sheets for the recipient list
4. Gmail for sending

### How I built it

I built the workflow one stage at a time and ran a test after each stage, so any problem could be traced to a single node.

#### a. Schedule Trigger

Added a Schedule Trigger node with two trigger rules, one at 8:00 AM and one at 8:00 PM.
Set the workflow timezone to Africa/Lagos so the times match local time.
Ran a test execution to confirm the trigger fired and passed 1 item on.

#### b. AI Agent with Google Gemini

Added an AI Agent node from the node panel.
Attached the Google Gemini Chat Model as the agent's model, using a Google AI API key credential.
Wrote the prompt in the agent's Prompt (User Message) field. It asks for one original quote of 25 words or fewer and switches between a morning and an evening style using $now.hour.
Executed the node and checked that it returned a single quote.

#### c. Google Sheet with the recipient list

Created a Google Sheet named Email List with a column of email addresses (the test addresses for this project).

#### d. Get row(s) in sheet

Added a Google Sheets node with the Get row(s) operation, which reads the emails from the sheet.
Executed it and confirmed it returned one item per row (3 items).

#### e. Gmail (Send a message)

Added a Gmail node with the Send operation and connected my Gmail account.
Mapped the fields to the outputs of the earlier nodes:
To: the email address from the current sheet row
Subject: a morning or evening title based on the time
Message (HTML): the quote from the AI Agent
Tested it and confirmed all 3 emails were sent.

#### f. Activate

Switched the workflow to Active so it runs on its own at 8 AM and 8 PM. The first scheduled run fired successfully.
Key expressions

#### What I learned
* How n8n passes items between nodes, and how one item becomes many.
* Using expressions to switch the AI prompt and the email wording by time of day.
* Why the workflow timezone matters for scheduled runs.
* Reading item counts on the connecting arrows to debug a workflow.
