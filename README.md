# SCADA Budget Tool

A web-based budgeting/quoting tool for SCADA projects. Engineers configure
licenses, hardware, and engineering mandays; the tool rolls it into a live
quote with a printable client-facing offer document.

**This version is one file.** No backend, no build step, no server to run
or pay for. GitHub Pages hosts `index.html`; GitHub itself is the database
(`data/users.json`, `data/pricing.json`, read and written directly from
your browser). That's the whole stack.

---

## Read this before you set it up

Dropping the backend is a real, deliberate tradeoff, not just a
simplification — please make sure it's the one you want:

- **Cost/margin visibility is a UI convention now, not a security
  boundary.** Every signed-in user's browser has the full pricing data
  (cost, margin, everything) in memory to make the app work at all. Sales
  not seeing cost is enforced by which columns render, not by what data
  exists in the page — anyone who opens dev tools can see it, regardless
  of role.
- **A read-only GitHub token lives in this file's source, in plain view.**
  Anyone who visits your GitHub Pages URL can extract it from the page and
  use it to *read* `data/users.json` and `data/pricing.json` directly —
  including everyone's password hash. It cannot write or delete anything;
  it only has read permission. Still, treat your repo's contents as
  effectively readable by anyone with the URL, not just people who log in
  through the app's UI.
- **Writing changes (saving pricing edits, managing users) needs a
  *different*, write-capable token**, which is never stored in this file.
  The Superadmin is asked to paste one the first time they save something
  in a browser tab; it's kept only in that tab (memory + `sessionStorage`)
  and forgotten when the tab closes.
- **Passwords are SHA-256 hashed, never stored in plaintext**, and the
  check happens in the browser. This is weaker than a server-side check
  (there's no rate limiting that actually holds up, and the hash itself is
  visible to anyone with the read-only token), but it's the ceiling for
  what a zero-backend setup can offer.

If any of this stops being acceptable later — say, once real customer
pricing is at stake and you want cost hiding to actually hold up — the fix
is adding back a small serverless proxy (Vercel, Cloudflare Workers,
whatever) that does the role filtering server-side. That's a contained
change to how data is fetched, not a rewrite of the app.

---

## Setup

### 1. Create a read-only GitHub token

1. **GitHub → Settings → Developer settings → Personal access tokens →
   Fine-grained tokens → Generate new token.**
2. Repository access: **Only select repositories** → the repo you're about
   to create/use for this app.
3. Repository permissions → **Contents: Read-only**.
4. Generate it, copy the token (`github_pat_...`).

### 2. Fill in the config at the top of `index.html`

Open the file (in GitHub's web editor, or any text editor), find these
lines near the top of the `<script>` section:

```js
const GH_OWNER = 'YOUR-GITHUB-USERNAME';
const GH_REPO = 'YOUR-REPO-NAME';
const GH_BRANCH = 'main';
const GH_READONLY_TOKEN = 'github_pat_PASTE_YOUR_READONLY_TOKEN_HERE';
```

Replace all four with your actual values.

### 3. Get it onto GitHub — entirely in the browser

1. github.com → **+ → New repository**. Name it, set **Private**, leave
   every checkbox unchecked (no README, no .gitignore).
2. On the new empty repo's page, click **"uploading an existing file."**
3. Drag in the one `index.html` file you just edited.
4. Commit.

### 4. Enable GitHub Pages

Repo → **Settings → Pages → Source: Deploy from branch → main → / (root)**.
Your app will be live at `https://<your-username>.github.io/<your-repo>/`
within a minute or two.

### 5. First-time setup

Open that URL. On the login screen, click **"First time using this repo?
Set it up →"**, fill in a username/password for the Superadmin account, and
click **Set Up**. It'll ask for a *write-capable* token (different from
the read-only one in step 1) —

- **GitHub → Settings → Developer settings → Personal access tokens →
  Fine-grained tokens → Generate new token**
- Repository access: this same repo only
- Repository permissions → **Contents: Read and write**

Paste it in. This one-time setup creates `data/users.json` (with your new
Superadmin account) and `data/pricing.json` (seeded from this page's
built-in default prices) directly in your repo — refresh the repo on
GitHub afterward and you'll see both files appear. Sign in with the
account you just created.

That write token isn't saved anywhere in the code — you'll be asked for it
again (once per browser tab) any time you save something as Superadmin
afterward.

---

## Roles

| Feature | Superadmin | Tendering Engineer | Sales |
|---|:---:|:---:|:---:|
| View prices | ✅ | ✅ | ✅ |
| Cost & margin columns shown | ✅ | ✅ | ❌ (hidden in the UI; see security notes above) |
| Build a quote (quantities/selections) | ✅ | ✅ | ✅ |
| Edit the master rate card | ✅ | ❌ | ❌ |
| User Management page | ✅ | ❌ | ❌ |

The built-in `Miguel` account can't be deleted or demoted from Superadmin
through the app's UI, so there's always a way back in. (Someone with a
write token could still edit `data/users.json` directly on GitHub to
override this — see security notes.)

---

## Managing users and pricing day-to-day

Sign in as any Superadmin and use **Admin → User Management** /
**Admin → Pricing Catalog** in the sidebar. Every save asks for your write
token (once per tab) and commits straight to the repo — check GitHub's
commit history on `data/users.json` / `data/pricing.json` any time for a
full audit trail of who changed what.

Prefer editing by hand instead? `data/users.json` and `data/pricing.json`
are just JSON files — open either on GitHub and click the pencil icon to
edit directly. Use the password-hash tool on the login screen ("Just need
a password hash for a manual edit? →") to generate a `passwordHash` value
without touching the app's save flow at all.

---

## Tech Stack

- **Pure HTML/CSS/JS**, one file, no framework, no build step, no
  dependencies to install
- **GitHub Pages** — static hosting, deploys on every push
- **GitHub Contents API**, called directly from the browser — storage,
  with full commit history
- **Web Crypto API** (`crypto.subtle`) — SHA-256 password hashing, built
  into every modern browser, nothing to import
