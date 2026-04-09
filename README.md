# claude-code-expense-plugin

A Claude Code plugin that fills out a Concur expense report for your monthly Claude Code subscription. Built for Outside Interactive employees.

> This is the first plugin in `qol-plugins` — a personal collection of quality-of-life Claude Code plugins. Sharing it because anyone on the team who expenses Claude Code every month can use it as-is.

You save the receipt PDF to a folder. You run one slash command. Claude drives Concur in a browser, fills out all form fields, attaches the PDF, and stops just before Submit so you can review and click **Submit Report** yourself.

No Gmail access. No OAuth. No scheduling. No corporate-IT friction.

## What it does

When you run `/submit-claude-expense` in Claude Code, it:

1. Finds the most recent unprocessed receipt PDF in `~/Documents/Outside/Reimbursement`
2. Extracts **Amount** and **Service Start Date** from the PDF using the pdf skill
3. Renames the file to the canonical form `Receipt-Claude-<Month><Year>.pdf`
4. Opens Concur in a Playwright-controlled Chrome browser (with persistent login)
5. Creates the expense report `Claude Code for <Month> <Year>` with Entity `Outside Interactive`
6. Adds a Software expense line item with Vendor Description `Anthropic - Claude Code subscription`, Service Schedule `Service (prepaid)`, and Amount/Transaction Date/Service Start Date filled from the PDF
7. Attaches the PDF as the receipt
8. **Stops before clicking Submit Report** — you review everything and click Submit yourself

### A note for non-Labs teammates

The slash command is pre-filled with the Labs team's Concur defaults for **Category** (`7000 - Foundational Services & Labs (360)`), **Brand** (`Foundational Services & Labs (FS&L)`), and **Department** (`235 - Labs`). On your first run, if Concur auto-populates these fields with different values for your team, the plugin will **stop and ask you** what to set them to instead of guessing. Tell it your team's correct values once and it'll fill them in for that run. If you want to make your team's values the permanent defaults, edit `claude-code-expense/commands/submit-claude-expense.md` (Step 6) and run `/reload-plugins`.

## File layout

This package is structured as a Claude Code **plugin marketplace** containing a single plugin. That's the pattern Claude Code requires for installing local plugins from disk.

```
claude-code-expense-plugin/                        # The marketplace folder (drop this anywhere)
├── .claude-plugin/
│   └── marketplace.json                           # Marketplace manifest (name: qol-plugins)
├── claude-code-expense/                           # The plugin itself
│   ├── .claude-plugin/
│   │   └── plugin.json                            # Plugin manifest (name: claude-code-expense)
│   ├── .mcp.json                                  # Registers the Playwright MCP server
│   └── commands/
│       └── submit-claude-expense.md               # The /submit-claude-expense slash command
└── README.md                                      # This file
```

## Prerequisites

- macOS
- [Claude Code](https://docs.claude.com/en/docs/claude-code) installed (`claude` in `$PATH`)
- Node.js 18+ (for `npx` to launch the Playwright MCP server)
- Access to Concur via your company's Google Workspace SSO

## Installation

### 1. Make sure the receipts folder exists

```bash
mkdir -p ~/Documents/Outside/Reimbursement
```

### 2. Add the marketplace to Claude Code

Open Claude Code from any directory:

```bash
claude
```

Then at the Claude Code prompt, add the marketplace directly from GitHub:

```
/plugin marketplace add portkeys/qol-plugins
```

Claude Code will clone the repo, read `.claude-plugin/marketplace.json`, and register the marketplace under the name **`qol-plugins`**.

### 3. Install the plugin from the marketplace

Still inside Claude Code:

```
/plugin install claude-code-expense@qol-plugins
```

The `@qol-plugins` suffix tells Claude Code which marketplace to install from. This enables the plugin, which:

- Registers the `/submit-claude-expense` slash command
- Auto-starts the Playwright MCP server from `.mcp.json`

You do NOT need to restart Claude Code. The slash command should be available immediately. If it isn't, run `/reload-plugins`.

To pick up future updates to this plugin, run `/plugin marketplace update qol-plugins` at the Claude Code prompt — Claude Code will pull the latest from GitHub.

### 4. First-time Concur login inside the Playwright Chrome profile

The Playwright MCP is configured to use a persistent Chrome user data directory at:

```
~/Library/Application Support/claude-code-playwright-profile
```

This is a **separate** Chrome profile from your everyday browser. You need to sign into Concur in it **once** so cookies are stored for subsequent runs.

Save any PDF (the current month's Anthropic receipt works, or any test PDF) to `~/Documents/Outside/Reimbursement` to give the first run something to process. Then at the Claude Code prompt:

```
/submit-claude-expense
```

When Playwright opens Chrome and lands on the Concur login page, **manually complete Google SSO** including 2FA. Once you're in, Claude will continue the workflow and cookies will be persisted to the profile. Future runs will reuse them.

If Concur SSO expires later (typically every 30–90 days), the next run will pause with a "Concur session expired" message — just sign in interactively once when that happens.

## Monthly workflow

From month two onward, the flow is:

1. Receipt email arrives from `invoice+statements@mail.anthropic.com` (typically the 4th of each month)
2. Click the attached PDF and save it to `~/Documents/Outside/Reimbursement` (any filename is fine)
3. Open a terminal and run:
   ```bash
   claude
   ```
4. At the Claude Code prompt:
   ```
   /submit-claude-expense
   ```
5. Watch Claude fill out Concur, review the draft report, click **Submit Report** in your browser when satisfied

Total user effort per month: ~30 seconds.

## Troubleshooting

**`/submit-claude-expense` isn't showing up after install**
Run `/reload-plugins` at the Claude Code prompt. If that doesn't work, check that the plugin is enabled with `/plugin` and look for `claude-code-expense` in the list.

**Marketplace add failed**
Make sure you can reach GitHub from your terminal (`gh auth status` or `git ls-remote https://github.com/portkeys/qol-plugins`). If you're behind a corporate proxy that blocks GitHub, you may need to clone manually and use a local path instead:
```bash
git clone https://github.com/portkeys/qol-plugins ~/claude-plugins/qol-plugins
```
Then inside Claude Code: `/plugin marketplace add ~/claude-plugins/qol-plugins`.

**"No PDF found in ~/Documents/Outside/Reimbursement"**
You haven't saved the receipt yet. Download the PDF attachment from the Anthropic receipt email and save it to that folder, then re-run.

**"A receipt for <Month> <Year> was already processed"**
A file named `Receipt-Claude-<Month><Year>.pdf` already exists in the folder. This is the idempotency guard — it assumes that month's Concur report was already created in a prior run. If you want to force re-processing, delete or rename the existing PDF first, then re-run.

**"Concur session expired"**
Sign into Concur in the Playwright Chrome window that opened, then re-run `/submit-claude-expense`. Your session will persist for the next several months.

**PDF parsing returned wrong values**
Open the PDF and check the Amount and Service Start Date. Update `claude-code-expense/commands/submit-claude-expense.md` with any field name adjustments the pdf skill needs. Run `/reload-plugins` after editing.

**Concur UI has changed (button moved, field renamed, new required field)**
Update the field list in Step 6 of `claude-code-expense/commands/submit-claude-expense.md` to match the new UI. The command is just a markdown file — edit it freely, then `/reload-plugins`.

**A run created a partial report in Concur but crashed**
Concur has no rollback. Delete the half-created report in Concur, then re-run `/submit-claude-expense`. The PDF may or may not have been renamed to the canonical form — if it was, rename it back to anything else (or delete the canonical file) before re-running, so the idempotency guard doesn't block you.

## Uninstalling

At the Claude Code prompt:

```
/plugin uninstall claude-code-expense@qol-plugins
/plugin marketplace remove qol-plugins
```

The saved receipt PDFs in `~/Documents/Outside/Reimbursement/` and the Playwright profile in `~/Library/Application Support/claude-code-playwright-profile` are left untouched. Delete those manually for a full cleanup.

## Contributing / editing the plugin

If you spot a bug (Concur UI changed, PDF parsing broke, etc.), open an issue or PR on [portkeys/qol-plugins](https://github.com/portkeys/qol-plugins). The slash command is just a markdown file at `claude-code-expense/commands/submit-claude-expense.md` — fixes are usually small text edits.

After a change is merged, everyone who has the marketplace installed can pick it up with `/plugin marketplace update qol-plugins`.
