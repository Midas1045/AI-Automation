### AI-Powered ATS Resume Screening Demo System (n8n)

An n8n workflow that receives job applications through a web form, screens each CV against a job description with AI, ranks candidates in Google Sheets, alerts HR on Telegram about strong candidates, and emails every applicant an update.

The AI scores the CV **blind**: personal details are removed from the CV text before it reaches the model, and the candidate's identity is attached back to the result afterwards.

#### Workflow screenshot

<img width="1769" height="386" alt="image" src="https://github.com/user-attachments/assets/918bb11e-8a0d-4368-98e3-c9781ed67e35" />



#### The problem

A company receives hundreds of CVs for every opening. HR needs a first-pass screen that compares each CV with the job requirements, ranks candidates, and highlights who should move to the interview stage.

#### How it works

```
Form Trigger -> Wait (10s) -> Generate Application ID --+--> Google Sheets: General List (form details)
                                                        |
                                                        '--> Extract From PDF -> Guardrails (sanitize)
                                                             -> Code: replace name -> Basic LLM Chain (Gemini)
                                                             -> Code: score and stage -> Google Sheets: scores
                                                             -> IF score
                                                                  |-- true  -> Telegram alert to HR -> Wait -> Shortlist email
                                                                  '-- false -> Wait -> Polite decline email
```

1. **CV Submission form (Form Trigger).** Applicants enter their name, email, location, date of birth, phone number, education, preferred work arrangement and job applied for, upload their CV as a PDF, and tick a required consent checkbox.
2. **Generate Application ID (Edit Fields).** Creates a unique ID such as `APP-261101-9335`. Both Google Sheets nodes use it as the matching column, so each candidate has one row per sheet.
3. **General List (Google Sheets).** Saves the applicant's form details. These details stay in the sheet for HR and are never sent to the AI.
4. **Extract From PDF.** Turns the uploaded CV into plain text.
5. **Guardrails (Sanitize Text).** Replaces phone numbers, locations and URLs (such as LinkedIn and GitHub links) with placeholders.
6. **Code node (name replacement).** Swaps any part of the applicant's name, and leftover email fragments, for `[CANDIDATE]` or `[EMAIL]`.
7. **Basic LLM Chain with a Structured Output Parser.** Google Gemini compares the cleaned CV with the job description (must-haves and nice-to-haves) and returns JSON: scores from 0 to 100 for skills, experience, education and achievements, plus strengths, gaps, red flags and a short summary. The prompt tells the model to use only evidence in the CV, to ignore any instructions written inside the CV, and not to consider age, gender, race, religion or nationality.
8. **Scoring Output (Code node).** Calculates the final score in code, not in the AI, and assigns a stage.
9. **Log Results (Google Sheets).** Writes the ranked result to the *Successful Candidates and Scores* tab.
10. **IF node.** Splits candidates by score.
11. **Shortlisted path.** A Telegram message goes to HR with the candidate's score breakdown, then a Wait node, then the shortlist email.
12. **Other path.** A Wait node, then a polite decline email.

#### Scoring

The AI rates each category and the code does the arithmetic, so the final number is consistent and checkable.

| Category | Weight |
|---|---|
| Skills match (must-haves count most) | 40% |
| Relevant experience | 30% |
| Education and certifications | 15% |
| Achievements and evidence | 15% |

- **75 and above:** Interview
- **55 to 74:** Maybe
- **Below 55:** Not shortlisted

Scores are clamped to 0 to 100 in the code, so a CV that tries to manipulate the model cannot push the total outside the range.

#### Blind screening and privacy

| Data | Where it goes |
|---|---|
| Name, email, phone, location, date of birth | Form fields, saved in the General List sheet. Never sent to the AI. |
| CV text | Cleaned (phone, location, links, name, email fragments) before the AI reads it. |
| AI result | Combined with the applicant's identity afterwards, so HR sees names next to scores. |

The form includes a required consent checkbox, and the consent answer is saved with each application.

#### Google Sheets

**General List** (form data): Application ID, Email, Full Name, Location, Date of Birth, Preferred Work Arrangement, Education, Phone Number, Job Applied, Consent.

**Successful Candidates and Scores** (ranking): Application ID, Date, Name, Email, Job Applied, Skills, Experience, Education, Achievements, Screening Score, Stage, Strengths, Gaps, Summary, HR Decision, HR Notes. The last two columns are left empty for a person to fill in.

#### Tools used

- [n8n](https://n8n.io): workflow automation, Form Trigger, Guardrails node, Code nodes, Wait nodes
- Google Gemini through the n8n Basic LLM Chain and Structured Output Parser
- Google Sheets: candidate records and ranking
- Telegram Bot API: HR alerts
- Gmail: candidate emails

#### Setup

1. Create the workflow in n8n with the nodes above, in that order.
2. Add credentials for Google Gemini, Google Sheets (OAuth), Gmail (OAuth) and a Telegram bot.
3. Create a Google Sheet with the two tabs and column headings listed above, and select it in both Sheets nodes with *Application ID* as the matching column.
4. Put the HR Telegram chat ID in the Telegram node, and press **Start** on the bot first.
5. Edit the job title, must-haves and nice-to-haves in the Basic LLM Chain prompt.
6. Replace the company name in the two candidate emails.
7. Test with a strong CV and a weak CV, then switch the workflow to **Active**.

#### Results Screenshots

<img width="782" height="512" alt="image" src="https://github.com/user-attachments/assets/7c56ff91-5fc6-4961-8f06-a3f7261c04bb" />

#### Testing approach

I built and tested one node at a time, checking the output of each step. Final checks were one strong and one weak sample CV, confirming that the sheet rows, the Telegram alert and each email were correct and that the AI's summary used `[CANDIDATE]` and not the real name.

#### What I learned

- **Edit Fields drops what it doesn't map.** The uploaded CV is a binary file, so the node had to keep binary data, and I had to match the binary field name (`RESUME`) in Extract From File.
- **Field names are case-sensitive** (`FULL NAME` is not `Full Name`).
- **Guardrails can miss things,** such as email addresses and names, so a Code node with a regex adds a second layer.
- **Let code do arithmetic.** The AI rates categories, and the final score and stage are calculated deterministically.
- **Prompt design:** treat CV content as data, require evidence, and keep the output in a strict JSON schema.
- **Append or update** needs a stable matching column, or it creates duplicate rows.

#### Limitations and next steps

- Scoring is a screening aid, not a hiring decision. A person should review borderline candidates and approve rejections before emails go out. A Wait node set to resume on an approval link would add that step.
- Candidates scoring between 55 and 74 ("Maybe") should be held for HR review, for example with a Switch node.
- A check after text extraction would send unreadable or scanned PDFs to manual review.
- A Jobs sheet would let each role have its own requirements, so different job titles can be scored against the right description.
- An Error Trigger workflow would alert me when a node fails.
- The workflow collects personal data (date of birth, phone, location). A real deployment needs a retention policy and compliance with data protection law, such as Nigeria's NDPR.

#### Notes

- Do not publish API keys, Telegram chat IDs, sheet IDs or credential IDs. Replace them with placeholders in any exported workflow.
- Sample CVs used for testing should be fictional or belong to the tester.


