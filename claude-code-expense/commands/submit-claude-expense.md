---
description: Read the most recent Claude Code receipt PDF from the Reimbursement folder and create a Concur expense report (stops before Submit for manual approval).
---

# Submit Claude Code Expense

You are filling out a Concur expense report for the user's monthly Claude Code subscription. The user has already saved the receipt PDF to a folder on disk — your job is to read it, extract two values, drive Concur in a browser to fill out the form, attach the PDF, and then STOP without submitting.

You do NOT have Gmail access. You do NOT need Gmail access. The PDF is already on disk.

## Start with a brief acknowledgement

Begin your response with this exact message:

> Looking for the latest Claude Code receipt in the Reimbursement folder…

## Step 1 — Find the receipt PDF

The receipts folder is:

```
~/Documents/Outside/Reimbursement
```

Use `ls -lt` (or equivalent) to list `.pdf` files in that folder by modification time, newest first.

Classify each PDF into one of two buckets:

- **"Processed"** — filenames matching the pattern `Receipt-Claude-<MonthName><Year>.pdf` (e.g. `Receipt-Claude-April2026.pdf`). These are receipts already processed in a prior run of this command. You should ignore them.
- **"Unprocessed"** — any other PDF filename (e.g. `Invoice-2701-9389-4927.pdf`, `statement.pdf`, or whatever name the user saved it under).

Pick the **most recently modified unprocessed PDF**. That's the one to process.

**Edge cases:**

- **Zero PDFs in the folder**: tell the user "No PDF found in `~/Documents/Outside/Reimbursement`. Please save the Anthropic receipt to that folder and re-run `/submit-claude-expense`." Then STOP.
- **Zero unprocessed PDFs, but one or more already-processed PDFs exist**: list the processed filenames and ask the user "These receipts look already processed. Do you want to force re-processing one of them? If so, tell me which filename." Then STOP and wait for an answer.
- **Multiple unprocessed PDFs**: pick the most recent by modification time, but tell the user which one you picked and list any others you're ignoring, so they can interrupt if you picked the wrong one.

## Step 2 — Extract data from the PDF

Use the `pdf` skill to extract these two fields from the selected PDF:

- **Amount** — the total charged (e.g. `$20.00` → `20.00`, as a plain number without the dollar sign)
- **Service Start Date** — the start of the billing/service period shown on the receipt, formatted as `MM/DD/YYYY`

The **Transaction Date** for Concur is the SAME value as Service Start Date. Don't extract it separately.

If you cannot confidently find both values in the PDF, STOP and report exactly what you see in the PDF to the user. Do not proceed to Concur with guessed values.

Also compute:

- **Month name** (e.g. `April`) and **4-digit year** (e.g. `2026`) from the Service Start Date. These are used for the report name and the target filename.

## Step 3 — Rename the PDF to the canonical filename (if needed)

Compute the canonical filename:

```
Receipt-Claude-<MonthName><Year>.pdf
```

Example: Service Start Date `04/04/2026` → `Receipt-Claude-April2026.pdf`

If the file is already named that, skip this step.

Otherwise, before renaming, check whether a file with the canonical name already exists in the folder:

- **Canonical filename already exists**: this means this month's receipt was already processed in a prior run. STOP and tell the user: "A receipt for <Month> <Year> was already processed (file `Receipt-Claude-<Month><Year>.pdf` already exists). A Concur report for this month was likely already created. If you want to re-process, delete or rename the existing file first." Do not create anything in Concur.
- **Canonical filename does not exist**: rename the unprocessed PDF to the canonical filename using `mv`. Report the rename to the user.

Record the full path to the PDF for use in Step 8. It must be an absolute path.

## Step 4 — Open Concur via the plugin's Playwright MCP

Use this plugin's bundled Playwright server — the `mcp__plugin_claude-code-expense_concur-chrome__*` tools, NOT the generic `mcp__playwright__*` server. Only the plugin's server drives real Chrome with the persistent user data directory that holds the Concur login session:

```
~/Library/Application Support/claude-code-playwright-profile
```

This profile persists Concur login cookies across runs.

Navigate to `https://www.concursolutions.com/` (or the tenant-specific URL if it redirects somewhere else).

**If you land on a login page**: the profile's Concur session has expired. Do NOT attempt to sign in yourself. STOP and tell the user: "Concur session expired — please sign into Concur in the Playwright browser window that just opened, then re-run `/submit-claude-expense`." Leave the browser window open so the user can sign in. The next run will reuse the freshly saved cookies.

Once signed in, navigate to **Expense → Manage Expenses → Create New Report** (or the current UI equivalent).

## Step 5 — Fill the "Create Expense Report" form

On the Create Expense Report form, fill these fields:

| Field | Value |
|---|---|
| Report Name | `Claude Code for <FullMonthName> <Year>` (e.g. `Claude Code for April 2026`) |
| Report Date | Leave as default (today) |
| Business Purpose | Leave blank |
| Entity | `Outside Interactive` (should be the default — verify it is selected; if not, select it) |
| Comment | Leave blank |

Click **Create Report**.

## Step 6 — Add the expense line item

On the newly created report, click **Add Expense** → **Manually Create Expense** (or "Create New Expense" if that's the current label).

On the "New Expense" form, fill these fields:

| Field | Value |
|---|---|
| Expense Type | `Software` |
| Transaction Date | Service Start Date from the PDF (MM/DD/YYYY) |
| Business Purpose or Comments | Leave blank |
| Vendor Description | `Anthropic - Claude Code subscription` |
| Amount | Amount from the PDF (number only, e.g. `20.00`) |
| Currency | `US, Dollar (USD)` — usually the default |
| Category | Leave as the auto-populated default (typically `7000 - Foundational Services & Labs (360)`) |
| Brand | Leave as the auto-populated default (typically `Foundational Services & Labs (FS&L)`) |
| Department | Leave as the auto-populated default (typically `235 - Labs`) |
| Project ID | Leave blank |
| Service Schedule | `Service (prepaid)` |
| Service Start Date | Same as Transaction Date (from the PDF) |

**If any of the auto-populated defaults (Category / Brand / Department) are missing, empty, or different from the defaults listed above, STOP and ask the user what to set them to.** Do not guess. (The defaults above are for the Labs team; users on other Outside teams will have different values and should tell you what to use.)

## Step 7 — Attach the PDF receipt

Use the Playwright MCP's file upload capability to attach the PDF as the receipt for this expense line item. Concur typically has an "Attach Receipt" or "Upload Receipt" button on the expense form; click it and upload the file.

Full path to upload (absolute — expand `~` to the user's home directory):

```
~/Documents/Outside/Reimbursement/Receipt-Claude-<MonthName><Year>.pdf
```

After attaching, click **Save Expense** to save the line item. **Saving the line item does NOT submit the report** — the report stays in "Not Submitted" state.

## Step 8 — STOP. Do not submit.

**CRITICAL REQUIREMENT**: Do NOT click "Submit Report". Do NOT click any button labeled "Submit". Leave the report in "Not Submitted" state so the user can review and submit it manually. This is explicit and non-negotiable.

## Step 9 — Report back

Summarize what you did in a short message:

- Which PDF you processed (filename, and whether you renamed it)
- Amount and Service Start Date extracted from the PDF
- The full path to the saved PDF
- The Concur report name and report number (if visible on the page)
- A clear reminder: "Please review in Concur and click **Submit Report** when ready."

Keep the summary short — a few lines. Don't restate the entire workflow.

## Failure handling

- **Canonical filename already exists**: STOP (covered in Step 3).
- **No PDF found in folder**: STOP (covered in Step 1).
- **PDF parsing fails or returns ambiguous values**: STOP. Do not create anything in Concur. Report exactly what you extracted and ask the user to fill the form manually this month.
- **Concur SSO expired**: STOP (covered in Step 4). Leave the browser open for the user to sign in.
- **Concur UI has changed** (button missing, unexpected modal, form field gone): STOP at the last step you successfully completed. Report exactly what worked and what broke. Do NOT force through or guess. The user will finish manually this time and update the slash command afterward.
- **Never retry automatically within a single run.** Fail clean.

## Context

- Company: Outside Interactive
- Concur Entity: Outside Interactive
- Receipts folder: `~/Documents/Outside/Reimbursement`
- Receipts typically arrive on the 4th of each month from `invoice+statements@mail.anthropic.com` with subject `Your receipt from Anthropic, PBC #XXXX-XXXX-XXXX`. The user manually saves the PDF attachment to the Reimbursement folder before running this command.
