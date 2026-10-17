# Goal Console

Daily, weekly, monthly, quarterly and yearly goals, each broken into tactics and
logged period by period, plus a vision board for the year. One HTML file, no
build step, no backend.

Data lives in an IndexedDB database in your browser. Google Drive is optional
and is what makes it the same console on every device.

---

## Deploy

```bash
git init
git add -A
git commit -m "Goal Console"
gh repo create goal-console --public --source=. --push
```

Then turn Pages on. Either route works:

**Actions** (this repo ships the workflow) — Settings → Pages → Source:
**GitHub Actions**. The next push deploys; later pushes redeploy automatically.

**Branch** (no YAML) — Settings → Pages → Source: **Deploy from a branch** →
`main` / `/ (root)`. Delete `.github/` if you go this way.

A minute later:

```
https://<username>.github.io/goal-console/
```

**The repo has to be public** unless you are on GitHub Pro or Team — Pages from
a private repo is a paid feature. Nothing here is sensitive: the file is the app
I wrote, your goals never touch it, and the Google client ID you add later is
not a secret.

`.nojekyll` is in the repo on purpose. Without it Pages runs the site through
Jekyll, which ignores files and folders beginning with an underscore.

---

## Connect Google Drive

Skip this and the app still works — it just keeps its data to whichever browser
you opened it in.

Drive needs an OAuth client that trusts your Pages address, and that client has
to live in your own Google Cloud project. That is what makes the data yours.

### 1. Google Cloud, once

At <https://console.cloud.google.com>:

1. **New project** — `goal-console`.
2. **APIs & Services → Library → Google Drive API → Enable.**
3. **APIs & Services → OAuth consent screen**
   - User type **External**
   - App name `Goal Console`, your email for support and developer contact
   - Scopes: add `.../auth/drive.file`
   - Test users: add your own Google address
   - Leave it in **Testing**. No verification needed — `drive.file` is a
     non-sensitive scope, and Testing allows 100 users.
4. **APIs & Services → Credentials → Create credentials → OAuth client ID**
   - Type **Web application**
   - **Authorised JavaScript origins**:
     ```
     https://<username>.github.io
     http://localhost:8000
     ```
     Scheme and host only. No path, no trailing slash — `/goal-console/` is a
     path and must not be included.
   - Leave **Authorised redirect URIs** empty. The app uses the token flow.
   - Copy the **Client ID**, ending `.apps.googleusercontent.com`.

### 2. In the app

Open your Pages URL, scroll to **Local database**, find the **Google Drive**
row. Paste the client ID, click **Connect Drive**, approve the prompt.

It creates `goal-console.json` in your Drive. On any other device: same URL,
same client ID, connect — your goals and vision board arrive.

---

## How the storage works

IndexedDB (`goal_console`, v2, stores `goals` / `vision` / `meta`) is the
working store, so the app is instant and works with no signal. Drive is the
shared copy the devices meet in.

- **Sync runs** on connect, on returning to the tab, and ~2.5s after a change.
- **Conflicts** resolve per record, newest edit wins. Two devices editing
  different goals both keep their work.
- **Deletions** travel as tombstones, so a deleted goal stays deleted rather
  than being pushed back by a device that still had it.
- **Offline** is fine. The next sync reconciles.

The scope is `drive.file` — the app can only touch files it created itself. It
cannot read or list anything else in your Drive. Revoke any time at
<https://myaccount.google.com/permissions>; your local copy survives.

Two other options sit alongside, in the same panel: **Database file** binds the
app to a real file on your machine and writes to it as you work (Chrome and Edge
only), and **Export / Import** moves the whole database as JSON.

---

## Local development

```bash
python3 -m http.server 8000
# http://localhost:8000
```

Drive needs an `http(s)` origin, so opening `index.html` by double-clicking
disables that one feature. Everything else works.

---

## Troubleshooting

| What you see | What it means |
|---|---|
| `redirect_uri_mismatch` / `origin_mismatch` | The origin is not in Authorised JavaScript origins, or you included the `/goal-console/` path. |
| `access_blocked`, "has not completed verification" | Your address is not in the consent screen's test users. |
| `idpiframe_initialization_failed` | Third-party cookies blocked for `accounts.google.com`. |
| "needs the page served over http(s)" | Opened as a file. Serve it, or use Database file instead. |
| Nothing happens on Connect | Popup blocker. Allow popups for your origin. |
| `error — access denied (403)` | Drive API not enabled, or the scope was not granted. Re-run consent. |
| 404 on the Pages URL | Pages not enabled yet, or still building — check the Actions tab. |

Updating later: replace `index.html`, commit, push. Data is in Drive and
IndexedDB, never in the page, so a redeploy loses nothing.
