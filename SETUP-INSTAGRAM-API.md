# Instagram Graph API — One-Time Setup for Hiraeth Haven

You only do this once. At the end you will have two strings to paste into GitHub Secrets:

- `IG_LONG_LIVED_TOKEN` — a 60-day access token (the script auto-refreshes it)
- `IG_USER_ID` — your @hiraethhavenshop Instagram Business Account ID

Total time: about 20 minutes.

---

## Step 1 — Confirm Instagram is set up correctly

Open the Instagram app on your phone:

1. Profile → menu (≡) → **Settings and privacy** → **Account type and tools**
2. Confirm it says **Business account** (or Creator). If not, switch.
3. Go to **Sharing to other apps** → **Facebook** → ensure @hiraethhavenshop is linked to your Hiraeth Haven Facebook Page.

If you don't have a Facebook Page yet, create one at facebook.com/pages/create — it can be empty, it just needs to exist for API authentication.

---

## Step 2 — Create a Meta Developer App

1. Go to https://developers.facebook.com and log in with the personal Facebook account that owns the Hiraeth Haven Page.
2. Top right → **My Apps** → **Create App**.
3. Use case: **Other** → **Next**.
4. App type: **Business** → **Next**.
5. App name: `Hiraeth Haven IG Feed` (only you see this). Contact email: HiraethHavenShop@gmail.com. **Create app**.

---

## Step 3 — Add the Instagram Graph API product

1. Inside your new app, scroll the left sidebar to **Add products**.
2. Find **Instagram Graph API** → **Set up**. (Not "Instagram Basic Display" — that one is being deprecated.)
3. Also add **Facebook Login for Business** (needed to generate the token).

---

## Step 4 — Get a short-lived user token

1. In the left sidebar: **Tools** → **Graph API Explorer**.
2. Top right "Meta App" dropdown: select `Hiraeth Haven IG Feed`.
3. "User or Page" dropdown: select **Get User Access Token**.
4. Click **Add a Permission** and check ALL of these:
   - `instagram_basic`
   - `instagram_manage_insights`
   - `pages_show_list`
   - `pages_read_engagement`
   - `business_management`
5. Click **Generate Access Token**. Log in and approve when Facebook prompts you.
6. Copy the token that appears in the box. **This expires in 1 hour** — you have to do steps 5 and 6 quickly.

---

## Step 5 — Find your Instagram Business Account ID

Still in Graph API Explorer, paste each query into the URL field and click **Submit**:

**5a. Get your Page ID:**
```
me/accounts
```
Find the entry for the Hiraeth Haven Page in the response. Copy its `id`.

**5b. Get your Instagram Business Account ID** (replace `<PAGE_ID>` with the value from 5a):
```
<PAGE_ID>?fields=instagram_business_account
```
The response gives you `instagram_business_account.id`. **This is your `IG_USER_ID`.** Save it.

---

## Step 6 — Exchange short-lived token for long-lived token

Open a terminal and run (replace the two placeholders):

```bash
curl -s "https://graph.facebook.com/v19.0/oauth/access_token?grant_type=fb_exchange_token&client_id=<YOUR_APP_ID>&client_secret=<YOUR_APP_SECRET>&fb_exchange_token=<SHORT_LIVED_TOKEN>"
```

- `<YOUR_APP_ID>` and `<YOUR_APP_SECRET>` are in the Meta dashboard under **App settings → Basic**.
- `<SHORT_LIVED_TOKEN>` is what you copied in step 4.

The response looks like:
```json
{"access_token":"EAAB...long string...","token_type":"bearer","expires_in":5183944}
```

That `access_token` is good for 60 days. **This is your `IG_LONG_LIVED_TOKEN`.** Save it.

The fetch script in this repo refreshes the token on every run, so as long as it runs at least once every 50 days, the token never expires.

---

## Step 7 — Add secrets to GitHub

1. Push the `instagram-gallery` folder to a GitHub repo (private is fine).
2. Repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**. Add three secrets:

| Name | Value |
|---|---|
| `IG_LONG_LIVED_TOKEN` | The long-lived token from step 6 |
| `IG_USER_ID` | The Instagram Business Account ID from step 5b |
| `IG_APP_SECRET` | Your Meta app secret (used to verify token refresh responses) |

Done. The cron workflow runs at 06:00 UTC daily and updates `data/instagram-feed.json` in the repo. Your Shopify section reads that file.

---

## Troubleshooting

**"Invalid OAuth access token"** — token expired. Repeat steps 4–6.

**"This endpoint requires the 'instagram_basic' permission"** — you missed a checkbox in step 4. Regenerate.

**Empty `data` array in the response** — the IG account has no posts, or the Business Account ID is wrong. Re-check step 5b.

**Rate limits** — IG Graph API allows 200 calls/hour per user. We make 2 calls per day. You'd have to break something pretty creatively to hit the limit.
