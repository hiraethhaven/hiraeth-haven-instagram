# Hiraeth Haven — Instagram Gallery

A daily-refreshed Instagram carousel that lives on hiraethhaven.com, pulls the latest 4 posts from @hiraethhavenshop via the Instagram Graph API, and stays well under any rate limits.

## Architecture

```
 ┌──────────────────┐    1×/day     ┌────────────────┐   commits   ┌─────────────────────────┐
 │ GitHub Actions   │ ───────────►  │ fetch-instagram│ ──────────► │ data/instagram-feed.json│
 │ cron @ 06:00 UTC │               │   .js (Node)   │             │  (in this repo)         │
 └──────────────────┘               └────────┬───────┘             └────────────┬────────────┘
                                             │                                  │ raw.githubusercontent.com
                                             │ 2 API calls/day                  ▼
                                             ▼                       ┌────────────────────┐
                                  ┌────────────────────┐             │ Shopify section    │
                                  │ Instagram Graph API│             │ instagram-feed.liq │
                                  │  (Meta)            │             │ — fetches JSON,    │
                                  └────────────────────┘             │   renders carousel │
                                                                     └────────────────────┘
```

**Why this design**

- **Zero browser-side API calls.** The Instagram token never leaves the GitHub runner. The store fetches a static JSON file — no CORS, no leaked secrets, no per-visitor rate limit.
- **Survives a busy day on the site.** The JSON is on GitHub's CDN. Even a viral spike does not cost you any Instagram API quota.
- **2 API calls per day total.** One token refresh, one media fetch. The Graph API limit is 200 calls/hour. We use 0.04% of it.
- **Auto-rotating token.** The Node script refreshes the long-lived token on every run, and (optionally) the workflow rewrites the GitHub secret. As long as the cron runs at least once every ~50 days the token never expires.

## Repo layout

```
instagram-gallery/
├── README.md                          ← you are here
├── SETUP-INSTAGRAM-API.md             ← one-time Meta/IG token setup
├── .github/
│   └── workflows/
│       └── instagram-feed.yml         ← daily cron + secret rotation
├── scripts/
│   ├── fetch-instagram.js             ← Node 18+ fetcher with token refresh
│   └── package.json
├── data/
│   └── instagram-feed.json            ← cached output (committed by the bot)
└── shopify/
    └── sections/
        └── instagram-feed.liquid      ← drop into theme/sections/
```

## Deployment — start to finish

### 1. Meta / Instagram setup (one time)

Follow `SETUP-INSTAGRAM-API.md`. At the end you'll have:

- `IG_LONG_LIVED_TOKEN`
- `IG_USER_ID`
- `IG_APP_ID` and `IG_APP_SECRET`

### 2. Push this folder to a new GitHub repo

```bash
cd "instagram-gallery"
git init
git add .
git commit -m "Initial Instagram gallery setup"
gh repo create hiraeth-haven-instagram --private --source=. --push
```

(Or use the GitHub website if you'd rather not use the CLI.)

### 3. Add the four secrets to the repo

Repo → **Settings → Secrets and variables → Actions → New repository secret**:

| Name | Value |
|---|---|
| `IG_LONG_LIVED_TOKEN` | from setup step 6 |
| `IG_USER_ID` | from setup step 5b |
| `IG_APP_ID` | Meta app dashboard → Settings → Basic |
| `IG_APP_SECRET` | Meta app dashboard → Settings → Basic |

**Optional but recommended** — for automatic token rotation:

| `GH_PAT_FOR_SECRETS` | a GitHub personal access token with `repo` scope. Needed so the workflow can rewrite `IG_LONG_LIVED_TOKEN` on itself when the API hands back a new token. Without this, you'll need to manually refresh the token every ~50 days. |

### 4. Run the workflow once manually

Repo → **Actions → Refresh Instagram Feed → Run workflow**. After ~30 seconds, `data/instagram-feed.json` should contain real posts.

Copy the **raw URL** of that file. It looks like:
```
https://raw.githubusercontent.com/<your-user>/hiraeth-haven-instagram/main/data/instagram-feed.json
```

### 5. Install the Shopify section

In your Shopify admin:

1. **Online Store → Themes → ⋯ → Edit code** on your live theme (or duplicate it first to be safe).
2. Open the `sections/` folder → **Add a new section** → name it `instagram-feed` → paste in the contents of `shopify/sections/instagram-feed.liquid`.
3. Save.
4. Go to **Customize** for the theme. On any page (Home is typical), click **Add section → Instagram Feed**.
5. In the section settings, paste your raw GitHub JSON URL into **Feed JSON URL**. Adjust the heading, colors, and post count to taste.
6. Save and preview.

### 6. (Optional) Cache the JSON through Shopify Files

If you'd rather not depend on raw.githubusercontent.com being reachable from every visitor's network:

- Shopify admin → **Content → Files → Upload** the JSON. Shopify gives you a `cdn.shopify.com` URL.
- Replace the section's Feed JSON URL with the Shopify CDN URL.
- Update the GitHub Action to upload the file via the Shopify Files API instead of (or in addition to) committing it. (Out of scope for v1 — committing to the repo is enough for >99% of stores.)

## Maintenance

- The cron runs daily at 06:00 UTC. Change the schedule in `.github/workflows/instagram-feed.yml`.
- If you ever rotate the IG password or change the linked Facebook Page, you must regenerate the long-lived token (repeat steps 4–6 of `SETUP-INSTAGRAM-API.md`) and update `IG_LONG_LIVED_TOKEN`.
- Adjust the number of posts in the Shopify section settings (range 2–8) — the script also accepts a `POST_LIMIT` env var if you want to cache more.

## API call budget

| When | What | Calls |
|---|---|---|
| Each cron run | Refresh token | 1 |
| Each cron run | Fetch latest media | 1 |
| Each visitor | (none — they hit GitHub's CDN) | 0 |
| **Daily total** | | **2** |

Instagram Graph API limit: **200 calls/user/hour**. We use 2/day = 0.083/hour.

## Brand styling notes

The section ships with Hiraeth Haven-friendly defaults — a warm cream background (`#f5efe6`), a muted antique brass accent (`#8a6a3b`), and serif typography inherited from your theme. All three colors are editable in the theme editor, so you can dial them to match your final palette without touching code.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Workflow runs but commits nothing | The JSON is unchanged (no new posts). Normal. |
| `Graph API 190: Invalid OAuth access token` | Long-lived token expired. Repeat setup steps 4–6 and update the secret. |
| Section shows skeleton then "Unable to load" | `feed_url` is wrong, the JSON file is private, or the GitHub repo is private and the raw URL needs auth. Make the repo public, or use Shopify Files (step 6 above). |
| Carousel doesn't scroll on iOS | Already handled by `-webkit-overflow-scrolling: touch`. If still broken, check that no parent has `overflow: hidden`. |
