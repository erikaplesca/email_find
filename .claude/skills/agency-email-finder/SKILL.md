---
name: agency-email-finder
description: Find and verify contact emails for agencies in General_Campaign_Emails.xlsx (outreach campaign). Use when asked to find, complete, fill in or continue agency emails, or to process the next N leads. Search the LinkedIn About section, then the agency website (homepage footer, Contact, About only). Verify ONE address per agency in TrueGuard. When TrueGuard has no credits, fall back to BounceZero, then to Hunter + Emailable + MyEmailVerifier (all three must pass). Only verified addresses go into the Excel file, written in bold.
---

# Agency email finder

Fill the empty **Email** column of `General_Campaign_Emails.xlsx` with verified contact emails for the agencies on the outreach list.

## THE GOLDEN RULE (never break this)

**No address goes into the Excel file unless TrueGuard returned exactly `Deliverable` for it in this run, or, only when TrueGuard has no credits left, the fallback verifiers passed it (see "Fallback verifiers").**
- `Undeliverable`, `Risky`, `Unknown`, catch-all, an error, a timeout or no verification at all all mean: **do not write it**.
- Never write an address you guessed or built from a pattern (info@, firstname@…). An address is a candidate only if you **saw it published** on LinkedIn or on the agency's own website.
- Never overwrite an email that is already in the file.
- **Do not guess ANYTHING.** If nothing passes, leave the cell empty. The user finds those manually.
- **One TrueGuard verification per agency, no more.** Pick the single best candidate, verify it, and accept the result. If it is not Deliverable, leave the cell empty and move on. No second attempt.

## Files and tools

| Thing | Where |
|---|---|
| Lead list | `General_Campaign_Emails.xlsx` in the project root (the folder Claude Code runs in), sheet `General_Campaign`. It is git-ignored: client data never goes to GitHub |
| Columns | A Company · **B Email** · C Company LinkedIn · D Country · E LinkedIn Members · F Score · G Why it is here · H Detail |
| Verifier | https://app.trueguard.io/project/email (box "Check an address" → **Validate**) |
| Fallback verifiers | Only after TrueGuard credits run out: BounceZero, then Hunter + Emailable + MyEmailVerifier (see "Fallback verifiers") |
| Browser | Claude in Chrome (the user is logged in to LinkedIn, TrueGuard and all fallback verifiers there) |
| Excel editing | Python with `openpyxl`. System Python does not have it: create a venv in the scratchpad (`python3 -m venv venv && venv/bin/pip install openpyxl`) |

## Step 0: Prepare

1. Load the Chrome tools (one ToolSearch call: tabs_context_mcp, tabs_create_mcp, navigate, computer, get_page_text, find, javascript_tool, read_page, tabs_close_mcp). Open one tab of your own and reuse it.
2. Back up the workbook to the scratchpad before the first write of the session.
3. Make the work queue: the rows where column A has a value and column B is empty, top to bottom. Process the batch size the user asked for (default: ask).
4. Open TrueGuard and read **Validation usage X / 100** (bottom left). Each Validate click uses 1 credit.

### When TrueGuard credits run out
The user does NOT want the paid plan, and never buys credits on any verifier. When usage reaches 100/100, or TrueGuard refuses a validation (limit reached, upgrade prompt):
- Tell the user TrueGuard is out of credits, and continue with the **fallback verifiers** below. Use only the sites listed there, never any other verifier.
- When the fallbacks are also out of credits: **STOP immediately.** Do not write any unverified address. Tell the user: "TrueGuard and the fallback verifiers are out of credits. Please log in with another account so I can continue." Report which row/company you stopped at.
- If you already found a candidate for the current agency but could not verify it, give it in the report as *not verified, not written*.
- Resume after the user confirms the new login. Check the usage counter again.

### Fallback verifiers (only when TrueGuard has no credits)
The user is logged in to all of them in Chrome. Check each one's remaining credits before you start. Use them in this order.

**1. BounceZero** (https://app.bouncezero.io/bulk-upload)
- Credits are shown in the pill at the top right; 1 credit per unique email.
- Click **Paste list**, type one address per line, click **Verify list**. Open the job (**Open in Bulk Results** → **View**) and read the EMAIL / SCORE / RESULT table.
- **Pass = `valid` with a score of 70 or more.** Only then write the address.
- `valid` with score 30 is a domain-level score: every address on that domain gets 30. Don't write it; list it in the report for the user to decide.
- `unknown` / `invalid` = do not write.

**2. When BounceZero is out of credits: Hunter + Emailable + MyEmailVerifier, all three for the same address**
- **Hunter**: open `https://hunter.io/verify/<email>`, then click the button whose text starts with "Verify" (each check costs 0.5 credit; "X remaining" is at the bottom left). The result reads `<email> is valid` or `<email> is invalid`. Hunter is lenient: it often says "valid" on catch-all domains, so a Hunter "valid" alone is never enough.
- **Emailable**: https://app.emailable.com/verifier. Type the address in the box and press Enter. Read the **State** (Deliverable / Risky / Undeliverable) and the score.
- **MyEmailVerifier**: https://client.myemailverifier.com/dashboard/welcome, in the **Quick check** box → **Verify** (1 credit; Unknown is free). Results arrive late and the previous result stays on screen, so wait until the result names the address you typed. Results: Valid / Catch-all / Invalid.
- **Pass = all three agree**: Hunter `valid` + Emailable `Deliverable` + MyEmailVerifier `Valid`. Only then write the address in bold.
- If only 2 of 3 pass, or any says Catch-all / Risky / Invalid: don't write it. Report it so the user can decide.
- In the run report, give one row per address with all three results (Hunter · Emailable · MyEmailVerifier · all 3?), plus the credits left on each tool.

All other rules stay the same: only published addresses (never guessed), one candidate per agency, and never overwrite an existing email.

## Step 1: LinkedIn About section

1. Open `<Company LinkedIn URL>/about/`.
2. Read the page with `get_page_text`. Look for an email in the Overview text and the Contact info.
3. Write down the **Website** field. You need it for Step 2.
4. If an email is there, it is a candidate. Still do Step 2 if it looks personal or odd, so you have alternatives.

## Step 2: The agency website

Visit ONLY these pages, in order, until you have at least one candidate:
1. **Homepage**: scroll to the bottom and read the **footer**.
2. **Contact** page (contact, contact-us, kontakt, contacto, yhteystiedot, kontakta-oss…).
3. **About** page (about, about-us, over-ons, om-oss, team…).

**Never use Privacy policy, Terms, Cookies, Imprint/Impressum or other legal pages.** Emails there are for data/legal requests, not for clients and business. Ignore any email you happen to see on them.

Fast extraction (run with `javascript_tool` on the agency's domain):
```js
const rx=/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/g;
const clean=a=>[...new Set(a||[])].filter(e=>!/\.(png|jpe?g|svg|webp|gif)$/i.test(e));
const out={here:clean(document.documentElement.innerHTML.match(rx)),
 mailto:[...document.querySelectorAll('a[href^="mailto:"]')].map(a=>a.href)};
for (const p of ['/contact','/contact-us','/about','/about-us']) {
  try { const r=await fetch(p); out[p]=r.status+' '+JSON.stringify(clean((await r.text()).match(rx))); } catch(e){ out[p]='err'; } }
out
```
- Use the real link names from the site's menu and footer when they differ from the default paths.
- **Cloudflare-hidden emails**: if a page contains `__cf_email__` / `data-cfemail` / `email-protection#`, decode them:
```js
const t=await (await fetch('/contact')).text(); // contact/about pages or homepage only
const dec=h=>{const k=parseInt(h.substr(0,2),16);let s='';for(let i=2;i<h.length;i+=2)s+=String.fromCharCode(parseInt(h.substr(i,2),16)^k);return s};
[...new Set([...t.matchAll(/data-cfemail="([0-9a-f]+)"|email-protection#([0-9a-f]+)/g)].map(m=>dec(m[1]||m[2])))]
```
- Also check for obfuscated text such as `name [at] domain.com`.
- If the website redirects (www → no www, another domain), call `tabs_context_mcp` again before running JS.
- Ignore addresses that belong to other companies (web agency credits, Sentry/Wix/example.com, image file names).

## Step 3: Choose the ONE candidate to verify

When there are several candidates on the agency's domain, pick a single one. Use this priority (general first):
1. General business inboxes: hello@, info@, contact@, hi@, mail@, studio@, team@, office@
2. Business/sales inboxes: newbusiness@, sales@, marketing@
3. A named person's address shown as the agency contact (founder, MD, new business)

Never pick privacy@, gdpr@, datacontroller@, legal@, jobs@, careers@ or support addresses for software products.
Verify only that one candidate (see the golden rule).

## Step 4: Verify in TrueGuard (mandatory)

1. Go to https://app.trueguard.io/project/email.
2. Click the "Check an address" box (placeholder `name@company.com`). Select all existing text (triple-click), type the candidate, click **Validate**.
3. Wait about 3–5 s, then zoom on the result card under the button.
4. Read the result title:
   - **Deliverable** (green) → accept the address and go to Step 5.
   - **Undeliverable / Risky / anything else** → reject it. Do NOT try another candidate. Leave the cell empty and go to the next agency.
5. Make sure the address in the result card is exactly the address you typed.
6. Faster reading: instead of a zoom, read the card with `javascript_tool`:
   `const t=document.body.innerText; const i=t.indexOf('Validate\n'); t.slice(i+9,i+120).split('\n').filter(Boolean).slice(0,3).join(' | ')` → `Deliverable | … | address`.
7. If a result could not be read (the page reloaded, or the text was wrong), **do not validate again**, because that costs another credit. Reload the page and find the address in the **Validations** table (`[...document.querySelectorAll('tr')]`), which keeps every result.
8. Before writing, re-check usage (`X / 100`). Stop at 100 (see "When TrueGuard credits run out").

## Step 5: Write to Excel (only Deliverable addresses)

- Write the address to column **B** of that row, in **bold**. Copy the existing font and set `b=True`, so the rest of the style stays the same.
- First check that column A of the row still has the company you expect and that B is empty.
- Close the file in Excel first (look for a `~$General_Campaign_Emails.xlsx` lock file). If it is open, ask the user to close it.
- Reload the file and print the row to confirm the write.

```python
import openpyxl, copy
p='General_Campaign_Emails.xlsx'  # run from the project root
wb=openpyxl.load_workbook(p); ws=wb['General_Campaign']
assert ws.cell(ROW,1).value==COMPANY and not ws.cell(ROW,2).value
c=ws.cell(ROW,2); c.value=EMAIL; f=copy.copy(c.font); f.b=True; c.font=f
wb.save(p)
```

## Step 6: No verified email

If there is no candidate, or the one verified candidate was not Deliverable: leave column B **empty**. Do not guess anything. The user fills these in manually. Record the company in the run report with the reason (e.g. "no email on LinkedIn, footer, contact or about", "hello@x.com → Undeliverable").

## Step 7: Run report (in chat)

After each batch, give the user a table: Row · Company · Where it was found · Candidates tried → TrueGuard result · Written (yes/no). Also give the TrueGuard credits used / remaining. List the empty rows so the user can do them manually. Close your browser tab.
