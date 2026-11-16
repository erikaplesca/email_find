# Setup guide

`setup · 01`

Everything you need to run **email_find** on your own machine, from an empty folder to the first verified email in bold.

It takes about 20 minutes the first time. Most of it is signing in to things.

---

## 1. What you need

| Thing | Why | Cost |
|---|---|---|
| A Mac or PC with **Python 3** | the workflow edits the Excel file with `openpyxl` | free |
| **Claude Code** | it runs the workflow (the skill in `.claude/skills/`) | Claude plan |
| **Google Chrome** + the **Claude in Chrome** extension | Claude reads LinkedIn, agency sites and the verifiers in your own browser | free |
| A **LinkedIn** account | for the company About pages | free |
| A **TrueGuard** account | the main verifier (app.trueguard.io) | free, 100 checks |
| **BounceZero**, **Hunter**, **Emailable**, **MyEmailVerifier** accounts | fallback verifiers, used only when TrueGuard is out of credits | free plans |

You don't need all the fallback accounts on day one. TrueGuard alone is enough to start.

## 2. Get the project

```bash
git clone <your-repo-url> email_find
cd email_find
```

You should see:

```
email_find/
├── README.md
├── docs/SETUP.md
├── docs/index.html
├── templates/leads-template.xlsx
└── .claude/skills/agency-email-finder/SKILL.md
```

## 3. Add your lead list

The workflow expects a file called **`General_Campaign_Emails.xlsx`** in the project root, with a sheet called **`General_Campaign`**.

```bash
cp templates/leads-template.xlsx General_Campaign_Emails.xlsx
```

Fill one agency per row, starting at row 2:

| Column | Header | What goes in it |
|---|---|---|
| A | Company | the agency name |
| B | **Email** | leave empty: this is what the workflow fills |
| C | Company LinkedIn | `https://www.linkedin.com/company/<slug>` |
| D | Country | e.g. `SE`, `NL`, `IE` |
| E | LinkedIn Members | optional |
| F | Score | optional, your own lead score |
| G | Why it is here | optional |
| H | Detail | optional |

The file is in `.gitignore`. It never goes to GitHub, and it shouldn't: it's client data.

*Keep the file closed in Excel while Claude works on it.* An open file leaves a `~$General_Campaign_Emails.xlsx` lock file, and the workflow will stop and ask you to close it.

## 4. Python

Nothing to install globally. The first time it writes to the Excel file, the workflow creates its own small virtual environment in Claude's scratch folder:

```bash
python3 -m venv venv && venv/bin/pip install openpyxl
```

If you'd rather have it in the project, run that line in the project root. `venv/` is git-ignored.

## 5. Chrome

1. Install **Claude in Chrome** from the Chrome Web Store and connect it to Claude Code.
2. In the same Chrome profile, sign in to:
   - LinkedIn
   - app.trueguard.io
   - (later) app.bouncezero.io, hunter.io, app.emailable.com, client.myemailverifier.com
3. When the extension asks for site permissions, allow these sites.

Claude never types your passwords. You sign in yourself; it only uses the sessions that are already open.

## 6. Run it

Open Claude Code in the project folder:

```bash
claude
```

Then ask in plain words:

```
process the next 20 leads
```

or

```
continue the agency emails from where we stopped
```

Claude loads the **agency-email-finder** skill and works top to bottom through the rows where column B is empty.

### What happens for each lead

1. **LinkedIn About** → an email if there is one, and the website.
2. **Website** → homepage footer, then Contact, then About. Nothing else, and never legal pages.
3. **One candidate** → general inbox first (`hello@`, `info@`, `contact@`), then sales inboxes, then a named contact.
4. **Verification** → see the next section.
5. **Excel** → only a verified address is written, in **bold**, in column B.

### The report

After every batch you get a table:

| Row | Company | Where it was found | Candidate → result | Written |
|---|---|---|---|---|

plus the credits left on each verifier, and the list of empty rows for you to do by hand.

## 7. Verification, in order

| Step | Verifier | How Claude uses it | Writes the address when |
|---|---|---|---|
| 1 | **TrueGuard** | "Check an address" → Validate | the result is exactly `Deliverable` |
| 2 | **BounceZero** | Bulk upload → Paste list → Verify list → Bulk Results | `valid` **and** score ≥ 70 |
| 3 | **Hunter + Emailable + MyEmailVerifier** | the same address in all three | **all three**: Hunter `valid`, Emailable `Deliverable`, MyEmailVerifier `Valid` |

Step 2 starts only when TrueGuard is out of credits. Step 3 starts only when BounceZero is out of credits too.

Never written, only reported:

- `Risky`, `Unknown`, `Catch-all`, `Undeliverable`, `Invalid`
- BounceZero `valid` with score **30**: that score belongs to the domain, so every address there gets it
- two out of three in step 3

## 8. Credits

All verifiers run on free plans. **No paid plans, ever.**

- **TrueGuard:** 100 checks per account. The counter is bottom left (`Validation usage X / 100`).
- **BounceZero:** the credit pill is top right. 1 credit per unique email.
- **Hunter:** 0.5 credit per check. "X remaining" is bottom left.
- **MyEmailVerifier:** 1 credit per check, and *unknown* results are free.
- **Emailable:** free credits on the account.

When every verifier is empty, Claude stops and says so. Log in with another free account, tell Claude, and it continues from the same row.

## 9. Picking up where you left off

The work is built to continue across days:

- **The Excel file is the source of truth.** Any row with column B empty is still to do.
- **Claude Code's memory** keeps a short progress note: the last row, credits left, which domains turned out to be catch-all. Start a new session with *"pick up where you left off"*.
- **The skill is versioned in git.** When you change a rule, commit it, and every future run follows the new version.

## 10. Troubleshooting

| What you see | What it means | What to do |
|---|---|---|
| LinkedIn shows errors after a few pages | rate limiting (HTTP 429) | Claude browses About pages one at a time; wait a few minutes |
| A site shows "Just a moment…" / a bot check | Cloudflare protection | Claude doesn't try to get past it. It looks for the address elsewhere (LinkedIn, registers, the Wayback Machine) |
| A search engine shows a CAPTCHA | too many searches | Claude stops using that engine. CAPTCHAs are always yours to solve, never Claude's |
| The website is gone | the agency closed or renamed | Claude checks the Wayback Machine and business registers for an old or new contact page |
| MyEmailVerifier shows the wrong address | results load late | Claude waits until the result names the address it typed |
| `~$General_Campaign_Emails.xlsx` exists | the file is open in Excel | close Excel and say "continue" |

## 11. Make it yours

Everything lives in one file: `.claude/skills/agency-email-finder/SKILL.md`.

- Change the lead file name or columns → the **Files and tools** table.
- Change which pages are allowed → **Step 2**.
- Change the verifier order or pass rules → **Fallback verifiers**.

Write rules in plain sentences. Claude follows them as written.

---

`setup · end`

What would you check twice before letting a system send it out?
