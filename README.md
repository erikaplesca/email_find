# email_find

this started as a spreadsheet with one empty column.

Three hundred agencies, each with a name, a LinkedIn page and a reason to be on my outreach list. What was missing was the one thing that actually lets you start a conversation: a real email address. So I built a small system with Claude Code that finds them *the slow, honest way*, and checks every single one before it goes anywhere.

No guessing. No `firstname@` patterns. Only addresses that people chose to publish.

---

## What it does

For every lead without an email, it:

1. **Reads the LinkedIn About section.** Sometimes the address is right there. It also notes the website.
2. **Visits the agency's own site.** Only the homepage footer, the Contact page and the About page.
3. **Picks one candidate.** A general inbox first (`hello@`, `info@`), a named person only when that's what the agency shows.
4. **Verifies it.** One check per agency, and the result decides.
5. **Writes it in bold** in the Excel file, but only if it passed. Otherwise the cell stays empty, for me to do by hand.

At the end of every batch there's a small report: what was found, where it came from, what the verifier said, and what's left.

## The rules I don't break

- Only addresses that were **published**: on LinkedIn, on the agency's website, or in a public register. Never built from a pattern.
- **Never** from privacy, terms or other legal pages. Those inboxes are for data requests, not for people.
- **One candidate per agency.** If it fails, it fails. The cell stays empty.
- **Never overwrite** an email that's already there.
- **No paid plans.** When the free credits run out, it stops and asks.

Easy feels productive. Careful is what actually works.

## How addresses get verified

| Order | Verifier | Passes when |
|---|---|---|
| 1 | TrueGuard | the result is exactly `Deliverable` |
| 2 | BounceZero (only when TrueGuard is out of credits) | `valid` with a score of 70 or more |
| 3 | Hunter + Emailable + MyEmailVerifier (only when BounceZero is out too) | **all three** agree: valid · Deliverable · Valid |

Anything else (risky, catch-all, unknown, two out of three) is reported, not written.

## Get started

The full guide lives in [`docs/SETUP.md`](docs/SETUP.md). There's also a designed version in [`docs/index.html`](docs/index.html) that you can publish with GitHub Pages.

The short version:

```bash
git clone <this repo>
cd email_find
cp templates/leads-template.xlsx General_Campaign_Emails.xlsx   # then add your leads
claude                                                          # open Claude Code here
```

Then, in Claude Code:

```
process the next 20 leads
```

## What's in here

```
email_find/
├── README.md                                   ← you are here
├── docs/
│   ├── SETUP.md                                ← the full setup guide
│   └── index.html                              ← the same guide, designed (GitHub Pages)
├── templates/
│   └── leads-template.xlsx                     ← empty lead list with the right columns
└── .claude/skills/agency-email-finder/
    └── SKILL.md                                ← the workflow Claude follows, step by step
```

The lead list itself (`General_Campaign_Emails.xlsx`) is **git-ignored**. It holds client data, and that stays on my laptop.

## brain notes

`brain notes · 01`

Small things I learned along the way:

- *unknown* on one address doesn't always mean the whole domain is a dead end. A second published address can come back valid.
- A score of 30 on BounceZero belongs to the domain, not the person. Every address there gets the same 30.
- Hunter is generous. It calls catch-all domains "valid", which is why it never decides alone.
- Some sites hide their email in ROT13. Some agencies rename and keep the same business ID. Some websites are gone, but the Wayback Machine still remembers the contact page.
- When a site shows a bot check or a CAPTCHA, the answer is to go around it honestly: another source, or a note for me. Never to break through it.

---

Built with Claude Code, a lot of curiosity and one stubborn spreadsheet.

What's the one task you keep doing by hand that you'd rather teach a system to do *carefully*?
