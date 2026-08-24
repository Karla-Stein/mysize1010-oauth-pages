# Google OAuth Production Setup — Step by Step

The clean path, based on what actually worked. Skip the dead ends (like
trying GitHub Pages' free subdomain for OAuth branding — it fails domain
ownership verification since you don't own the root domain).

## Part 1 — Get a real, owned domain

1. Buy a cheap domain (~£1-10/year with a new-customer promo, any
   registrar). Don't buy multiple TLDs — one `.com` is enough.
2. In the registrar's DNS settings, add these records to point the
   domain at GitHub Pages:
   - 4× A records, host `@`, values:
     `185.199.108.153`, `185.199.109.153`, `185.199.110.153`,
     `185.199.111.153`
   - 1× CNAME record, host `www`, value: `<your-github-username>.github.io`
3. Leave any existing mail-related records (MX, TXT, DKIM, DMARC)
   untouched — they're unrelated.

## Part 2 — Host the OAuth pages on GitHub Pages

1. Create a GitHub repo (public), no need to open it in VS Code — the
   browser upload interface is enough.
2. Add two files inside a project-specific folder (e.g. `/mysize1010/`):
   - `index.html` — plain homepage describing what the app does
   - `privacy.html` — privacy policy: what data is accessed, how it's
     used, how to revoke access
3. In repo Settings → Pages → Custom domain, enter your domain
   (e.g. `opsatelierhq.com`). Wait for "DNS check successful."
4. Enable "Enforce HTTPS" once it becomes available.

## Part 3 — Verify domain ownership in Google Search Console

1. Go to Search Console → Add property → **URL prefix** (not "Domain" —
   URL prefix lets you verify a specific path).
2. Enter the exact homepage URL (e.g.
   `https://opsatelierhq.com/mysize1010/`).
3. Choose **HTML file** verification. Download the file Google gives you.
4. Upload that exact file (same filename, no renaming) to the same
   folder in the GitHub repo. Confirm it loads directly in a browser
   before clicking Verify.
5. Click Verify in Search Console.

## Part 4 — Fix the Google Cloud OAuth consent screen

1. In Google Cloud Console → OAuth consent screen → Branding:
   - App name: must exactly match the visible heading on the homepage
   - Application home page: the GitHub Pages URL from Part 2
   - Application privacy policy link: the privacy.html URL
   - Authorized domains: your bare domain (e.g. `opsatelierhq.com`,
     no `https://`, no path)
2. Confirm your Google account has **Owner or Editor** role on this
   specific Cloud project (IAM & Admin → IAM) — domain verification
   only counts if done by an account with that role on the project.
3. Once Search Console shows verified, request branding re-verification
   in Cloud Console. It can take some time to propagate — if it still
   shows unverified shortly after, wait and recheck rather than
   re-troubleshooting from scratch.
4. Publish the app: OAuth consent screen → change publishing status
   from **Testing** to **In production**. This is the step that
   actually removes the 7-day refresh token expiry — everything above
   exists to make this step possible.

## Part 5 — Fix it on the automation platform (n8n or similar)

This is the step most likely to be missed, and the one that caused
today's real confusion.

1. **List every credential your workflow actually uses** — a workflow
   can have several Google credentials scattered across different
   nodes (Sheets, Drive, Slides, etc.), not just one.
2. For each one, check whether it's on **Managed OAuth** (the
   platform's own pre-verified app — never has the 7-day problem, but
   also isn't tied to your custom app) or **Custom OAuth** (your own
   Google Cloud app — this is the one Part 4 actually protects).
3. In Google Cloud Console, enable every Google API the workflow
   actually calls (Drive, Sheets, Slides, etc.) — not just the one you
   happened to be debugging.
4. Reconnect every node's credential to the same single verified
   Custom OAuth app from Part 4. Don't leave some nodes on Managed and
   others on Custom — pick one, consistently.
5. If any file the workflow touches is owned by a different Google
   account than the one you're authenticating as, that file (or its
   containing folder) must be explicitly shared with your account as
   Editor. Ownership doesn't transfer just because a workflow touches
   the file.
6. Test-run the workflow before publishing any change to production.

## The core lesson

A single reconnect looks like it fixes the problem, then it comes back,
because the underlying app was never actually taken out of Testing
status, or because only one of several credentials got fixed while
others stayed on the old, expiring setup. All five parts have to be true
at once — branding verified, app published, right APIs enabled, every
node on the same verified credential, and file access actually shared.
