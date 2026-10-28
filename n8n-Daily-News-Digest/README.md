## n8n Daily News Digest

An automated n8n workflow that reads the latest Nigerian headlines, uses AI to write a short briefing, and delivers it twice a day, every day at **8:00 AM** and **8:00 PM**, by **Telegram** and by a styled **HTML email** sent to everyone on a Google Sheet list.



### How it works

```
Schedule Trigger -> Daily Post RSS Read -> Aggregate -> AI Agent -> Edit Fields
                        20 items            1 item       1 item        |
                                                                       |-> Telegram Message
                                                                       |
                                                                       '-> Code (build HTML) -> Email List (Google Sheets) -> Gmail
                                                                              1 item                3 items                    3 emails
```

1. **Schedule Trigger** runs the workflow at 8:00 AM and 8:00 PM using two rules.
2. **Daily Post RSS Read** pulls the latest stories from the Daily Post Nigeria hot-news RSS feed (20 items).
3. **Aggregate** combines the 20 stories into one list, so the AI runs once, not once per story.
4. **AI Agent** (OpenAI chat model) receives the top stories and writes a briefing of the 8 most important ones. The prompt keeps it factual: it may only use the information in the feed, must write in its own words, and must follow a fixed output format.
5. **Edit Fields** copies the briefing into a `data` field for the next steps.
6. The workflow then splits into two branches:
   - **Telegram Message** sends the briefing to a Telegram chat, with Parse Mode set to HTML so bold text and links display properly.
   - **Code node ("Javascript code for Header Block")** turns the same briefing into a styled HTML email, with a header block, a greeting that changes with the time of day, and one card per story. **Email List** (Google Sheets) then reads the recipients, and **Gmail message** sends one email per recipient.

### Tools used

- [n8n](https://n8n.io) (workflow automation)
- OpenAI chat model through the n8n AI Agent node
- Daily Post Nigeria RSS feed (news source)
- Telegram Bot API (instant delivery)
- Google Sheets (recipient list)
- Gmail (email delivery)

### What the output looks like

**Telegram message**

> Good morning!
> Here's today's news
> **Top stories: elections, security, economy and public health**
>
> • **Headline**: short summary. Read more

**Email:** a dark header block with the greeting and the day's themes, followed by a white card for each story with a gold stripe and a "Read more" link. A footer says the summaries are written by AI from Daily Post Nigeria headlines.

### Key parts of the setup

**Schedule:** two rules, one at hour 8 and one at hour 20. Set the workflow timezone to your own (this project uses Africa/Lagos) so the runs and the "Good morning / Good evening" greeting match local time.

**Combining stories for the AI** (inside the AI Agent prompt):

```
{{ $json.data.slice(0, 12).map(i => '- ' + i.title + ' | ' + i.link + ' | ' + i.contentSnippet).join('\n') }}
```

This turns the aggregated list into plain text, one story per line, for the model to read.

**Greeting that follows the time of day:**

```
{{ $now.hour < 12 ? 'Good morning!' : ($now.hour < 17 ? 'Good afternoon!' : 'Good evening!') }}
```

**Gmail body:** the email reaches back to the Code node to get the finished HTML:

```
{{ $('Javascript code for Header Block').item.json.html }}
```

`.item` works here because the Code node produces one item and the Google Sheets node fans it out to many recipients, so every recipient is linked back to that same item.

### Setup

1. Create the workflow in n8n and add the nodes in the order shown above.
2. Add your own credentials: an OpenAI API key, a Telegram bot token (from @BotFather), Google Sheets (OAuth), and Gmail (OAuth).
3. Send your Telegram bot a message first (press **Start**), then put your numeric Telegram chat ID in the Telegram node.
4. Create a Google Sheet named **Email List** with the recipients' addresses in a column, and match the column name in the Gmail node's **To** field.
5. Set the workflow timezone, test each node in turn, and then switch the workflow to **Active**.

### Testing approach

I built the workflow one node at a time and ran a test after each step, checking the number of items on every connection (1 → 20 → 1 → 1 → 3). That made it easy to find where something broke. Before the first full run, I tested the email branch with only my own address in the sheet.

### What I learned

- How n8n passes items between nodes, and how one item becomes many (and the reverse, with Aggregate).
- Writing prompts that keep an AI model to the facts and produce a strict output format.
- Why Telegram and HTML email need different formatting, and how a Code node can convert one into the other.
- Using expressions to change content by time of day.

### Limitations and next steps

- Telegram rejects messages over 4,096 characters, so the prompt caps the briefing length.
- The briefing relies on one source. Adding more feeds (for example BBC or Al Jazeera) would widen the coverage.
- Possible improvements: error alerts if a node fails, logging each run to a sheet, an unsubscribe option, and skipping stories that were already sent.

### Notes

- Summaries are generated by AI from public RSS headlines and snippets. Each item links back to the original story, and the source is credited in the email footer.
- Credentials are not stored in this repository. Any workflow export should have the sheet ID, chat ID, and credential IDs replaced with placeholders before sharing.



