# Dashboard

Three consoles, one site, one deployment.

```
/                 landing page, links to all three
/goal/            Goal Console
/property/        Property Console
/budget/          Budget Console
```

| Page | What it is |
|---|---|
| **Goal Console** | Monthly, quarterly and yearly goals, each with the daily and weekly systems that produce it, plus a vision board. |
| **Property Console** | The rental portfolio month by month: income, costs, cashflow, return on cash and gross yield, per property. |
| **Budget Console** | A household month: budget against actual by category, who spent what, and savings pots filling up. |

Every console links back to the landing page from its header. One HTML file
each, no build step, no backend.

Data lives in an IndexedDB database in your browser — a separate one per page
(`goal_console`, `property_console`, `budget_console`), so none of them tread
on each other.
Google Drive is optional and is what makes them the same consoles on every
device.

---

## Deploy

```bash
git init
git add -A
git commit -m "Dashboard"
gh repo create dashboard --public --source=. --push
```

Then turn Pages on. Either route works:

**Actions** (this repo ships the workflow) — Settings → Pages → Source:
**GitHub Actions**. The next push deploys; later pushes redeploy automatically.

**Branch** (no YAML) — Settings → Pages → Source: **Deploy from a branch** →
`main` / `/ (root)`. Delete `.github/` if you go this way.

A minute later:

```
https://<username>.github.io/dashboard/          # landing page
https://<username>.github.io/dashboard/goal/     # goals
https://<username>.github.io/dashboard/budget/   # budget
```

**The Pages URL follows the repo name.** Rename the repo and every path moves
with it; GitHub redirects the old name, but update your bookmarks and the git
remote (`git remote set-url origin …`) anyway.

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

Each page keeps its own file — `goal-console.json`, `property-console.json`
and `budget-console.json` —
and each needs the client ID pasted in once. **One OAuth client covers both**:
they share an origin, so the client ID you set up for the goals page works
unchanged on the budget page. On any other device: same URL, same client ID,
connect — your data arrives.

---

## How the storage works

IndexedDB is the working store, so each page is instant and works with no
signal; Drive is the shared copy the devices meet in.

| Page | Database | Stores |
|---|---|---|
| Goals | `goal_console` v2 | `goals`, `vision`, `meta` |
| Property | `property_console` v1 | `records`, `meta` |
| Budget | `budget_console` v1 | `records`, `meta` |

Both follow the same rules:

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

Updating later: replace any of the `index.html` files, commit, push.
Data is in Drive and IndexedDB, never in the page, so a redeploy loses nothing.

Moving the pages around is safe too: **IndexedDB is keyed by origin, not by
path**, so everything already saved in your browser survives a repo rename or a
folder move, as long as the host stays `<username>.github.io`. The Google OAuth
client is keyed by origin for the same reason — scheme and host only, no path —
so a rename needs no change in Google Cloud.

## Budget Console notes

- **Categories** carry a standing monthly budget. Editing one offers *This
  month only*, which overrides the figure for the month in view and leaves the
  standing budget alone.
- **Entries** are tagged with who paid, which drives the *Who spent it* split.
- **Deleting a category** deletes its entries too, so no spending is left
  orphaned and invisible.
- **Pots** run across every month; the balance is the sum of every payment in.
  A negative amount records a withdrawal.
- Colours: the category meters use the reserved under / watch / over status
  triad; Ope and Justina get their own validated hue each, kept clear of those
  three so a colour never means two things at once.

## Property Console notes

Standing figures live on the property: rent, mortgage, insurance, management
as a percentage of rent, current value and the cash invested. Those recur every
month on their own. What varies is logged per month.

- **Rent this month** on the card overrides the standing rent for that month
  alone — set it to 0 for a void. The card flags a void or a part month, and
  **Reset** puts the standing figure back.
- **One-off costs** are logged against the month with a category.
- **Deleting a property** deletes its costs too, so nothing is left orphaned.

The two return figures, spelled out so there is no ambiguity:

| Figure | How it is worked out |
|---|---|
| **Cashflow** | rent received − (mortgage + insurance + management + one-off costs) |
| **Return on cash** | cashflow × 12 ÷ cash invested — this month annualised, so a month with a big repair reads honestly low |
| **Gross yield** | standing rent × 12 ÷ current value |

Both are arithmetic on the figures you enter, nothing more. They take no
account of tax, capital growth, or periods between tenancies beyond the months
you mark as voids.

In the year chart, months that have not happened yet are the standing figures
repeating — they are drawn faint, labelled as projections, and left out of the
"to date" total.
