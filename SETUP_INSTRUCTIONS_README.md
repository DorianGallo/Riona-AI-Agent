# Riona AI Agent — Cookie Setup & Session Recovery Guide

## Project Reference
- **Project root:** `C:\Users\doria\RionaAIpoptart\Riona-poptart-Agent`
- **Cookies file:** `C:\Users\doria\RionaAIpoptart\Riona-poptart-Agent\cookies\Instagramcookies.json`
- **Shell:** Git Bash (Windows)

---

## Background

Riona uses Puppeteer to automate Instagram. Instagram's bot detection blocks automated credential logins. The solution is to **manually log in via Playwright**, capture the session cookies, convert them to the format Riona expects, and place them at the exact path Riona reads from.

Riona reads cookies from **one hardcoded path only:**
```
<project-root>/cookies/Instagramcookies.json
```
The filename is case-sensitive: capital `I`, capital `c`.

---

## How to Know You Need to Reset Cookies

Any of these symptoms means you need to follow this guide:

| Terminal Output | Cause |
|---|---|
| `Cookies file does not exist` | File was deleted or never created |
| `Login with credentials failed` | Instagram blocked automated login |
| `Detached Frame` error | Login failed mid-retry, browser crashed |
| `Cookies are invalid or expired` | Session expired (~90 days) or IP changed |
| `Checking cookies existence: false` | File is empty, missing, or corrupt |
| Agent starts but does nothing | Cookies loaded but session is no longer valid |

---

## The Full Recovery Process

---

### STEP 1 — Navigate to the project root

Open Git Bash and run:

```bash
cd /c/Users/doria/RionaAIpoptart/Riona-poptart-Agent
```

Confirm you are in the right place:

```bash
pwd
```

Expected output:
```
/c/Users/doria/RionaAIpoptart/Riona-poptart-Agent
```

---

### STEP 2 — Delete the old cookies file

```bash
rm -f cookies/Instagramcookies.json
```

Also delete any corrupt backup files Riona may have created:

```bash
rm -f cookies/Instagramcookies.corrupt-*.json
```

Confirm the folder is clean:

```bash
ls cookies/
```

Expected output: blank (nothing listed) or the folder is empty.

---

### STEP 3 — Recreate the cookies folder if it is missing

Only needed if `ls cookies/` gave `No such file or directory`:

```bash
mkdir -p cookies
```

---

### STEP 4 — Capture fresh cookies with Playwright

Run this command. A Chromium browser window will open:

```bash
npx playwright codegen --save-storage=cookies/Instagramcookies.json https://www.instagram.com/accounts/login/
```

**In the Chromium browser window:**

1. Type your Instagram username manually
2. Type your Instagram password manually
3. Click **Log in**
4. Complete any **2FA / verification code** if Instagram asks
5. Wait until you can fully see the **Instagram home feed** (the main scroll of posts) — do not stop early
6. **Close the Chromium browser window** by clicking the X

Closing the browser triggers Playwright to save the cookies automatically.

---

### STEP 5 — Convert the cookies to the format Riona expects

Playwright saves cookies in its own storage state format (an object with `cookies` and `origins` keys). Riona expects a **plain JSON array**. Run this conversion:

```bash
node -e "
const fs = require('fs');
const raw = JSON.parse(fs.readFileSync('./cookies/Instagramcookies.json', 'utf8'));
const cookies = raw.cookies || raw;
fs.writeFileSync('./cookies/Instagramcookies.json', JSON.stringify(cookies, null, 2));
console.log('Done! Converted', cookies.length, 'cookies.');
"
```

Expected output:
```
Done! Converted 9 cookies.
```

---

### STEP 6 — Verify the cookies are valid

```bash
node -e "
const c = require('./cookies/Instagramcookies.json');
const s = c.find(x => x.name === 'sessionid');
const now = Math.floor(Date.now() / 1000);
console.log('Is array    :', Array.isArray(c));
console.log('Cookie count:', c.length);
console.log('sessionid   :', s ? 'FOUND' : 'MISSING');
console.log('Expires     :', s ? new Date(s.expires * 1000).toISOString() : 'N/A');
console.log('Valid       :', s && s.expires > now ? 'YES' : 'EXPIRED');
"
```

**Expected output:**
```
Is array    : true
Cookie count: 9
sessionid   : FOUND
Expires     : 2026-09-xx...
Valid       : YES
```

**If you see `sessionid: MISSING` or `Valid: EXPIRED`** — go back to Step 4 and repeat. Make sure you are fully on the home feed before closing the browser.

---

### STEP 7 — Start the agent

```bash
npm start
```

**Expected terminal output (success):**
```
info: Server is running on port 3000
info: MongoDB connected
info: Loaded cookies. Navigating to Instagram home page.
info: Successfully logged in with cookies.
Liking post 1...
```

---

## Quick Reference Cheatsheet

Once familiar with the process, these are the only commands you need every time:

```bash
# 1. Go to project root
cd /c/Users/doria/RionaAIpoptart/Riona-poptart-Agent

# 2. Delete old cookies
rm -f cookies/Instagramcookies.json cookies/Instagramcookies.corrupt-*.json

# 3. Capture fresh cookies (log in manually in the browser, then close it)
npx playwright codegen --save-storage=cookies/Instagramcookies.json https://www.instagram.com/accounts/login/

# 4. Convert format
node -e "const fs=require('fs');const raw=JSON.parse(fs.readFileSync('./cookies/Instagramcookies.json','utf8'));const c=raw.cookies||raw;fs.writeFileSync('./cookies/Instagramcookies.json',JSON.stringify(c,null,2));console.log('Converted',c.length,'cookies.');"

# 5. Verify
node -e "const c=require('./cookies/Instagramcookies.json');const s=c.find(x=>x.name==='sessionid');const now=Math.floor(Date.now()/1000);console.log('Valid:',s&&s.expires>now?'YES':'NO');"

# 6. Start
npm start
```

---

## Important Notes

### On IP / Proxy changes
Instagram sessions are tied to the IP address used during login. If your IP changes (new network, VPN on/off, proxy rotation), the existing `sessionid` cookie becomes invalid. You must re-capture cookies from the new IP every time this happens. Always run the Playwright capture from the same network/IP that Riona will use when running.

### On cookie expiry
Instagram `sessionid` cookies typically last around 90 days. After that the session is automatically invalidated and you will see the `Cookies are invalid or expired` warning. Follow the full guide above to get a fresh session.

### On Instagram security alerts
If Instagram sends you a security alert email or SMS after you log in via Playwright, confirm it is you. Ignoring the alert may cause Instagram to invalidate the session sooner than expected.

### Never commit the cookies file to Git
The `cookies/` folder is already listed in `.gitignore` in the Riona project. Never manually add or push `Instagramcookies.json` to any repository — it contains your active session token.

### Why automated credential login fails
Riona uses Puppeteer (a headless browser). Instagram detects the browser fingerprint as a bot and blocks or challenges the login. Playwright's `codegen` tool opens a browser that more closely mimics a real user, which is why the manual capture method works reliably.

---

## Troubleshooting

| Problem | Solution |
|---|---|
| `npx playwright codegen` command not found | Run `npm install playwright` then `npx playwright install chromium` |
| Browser opens but Instagram shows CAPTCHA | Complete the CAPTCHA manually, then continue logging in |
| `Done! Converted 0 cookies` | The browser was closed before login completed — repeat Step 4 |
| `sessionid: MISSING` after conversion | You were not fully logged in when you closed the browser — repeat Step 4 |
| `Valid: EXPIRED` immediately after capture | System clock may be wrong, or Instagram rejected the session — repeat Step 4 |
| Agent logs in but does nothing | Check that `IG_AGENT_ENABLED=true` is set in your `.env` file |
| `Comment box not found` in logs | Non-critical — Instagram changed a UI selector. Agent continues running |