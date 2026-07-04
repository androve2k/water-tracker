# 💧 Water Tracker

**🔗 Live demo: [roversia.it/apps/acqua.html](https://roversia.it/apps/acqua.html)**

A small web app for tracking daily water intake, with a custom goal, streaks, and a weekly chart. Auth is **username + PIN** (no email), data synced via Firebase Realtime Database.

Part of the app suite on [roversia.it](https://roversia.it/apps.html), my personal site — built entirely in **vanilla HTML/CSS/JS, zero frameworks, zero build step**.

> Code comments are in Italian (my working language) — happy to translate any part on request. A full write-up of the site and its architecture is on my [blog](https://roversia.it/blog.html) (Italian).

---

## Tech stack

- **Frontend:** plain HTML + CSS + JavaScript (no dependencies, no bundler)
- **Backend:** Firebase Realtime Database via **REST API** (no Firebase SDK)
- **Auth:** username + PIN, **PBKDF2-SHA256** hash (150,000 iterations) computed client-side with the Web Crypto API — the PIN never leaves the browser in plain text
- **Session:** `localStorage`, 30-day expiry
- **Atomic writes:** **ETag + `If-Match`** pattern with up to 4 retries on conflict (HTTP 412), to avoid race conditions when the same user writes from multiple tabs/devices
- **Optimistic UI:** the interface updates immediately on tap, then confirms (or rolls back) based on the server response

## Data structure (Realtime Database)

```
apps_users/
  {username}/ { s: saltB64, h: hashB64, c: timestamp }

apps_water/
  {username}/
    goal: 2000
    days/
      2026-07-02: 1750
```

## RTDB security rules

```json
{
  "rules": {
    "apps_users": {
      "$u": {
        ".read": true,
        ".write": "!data.exists()",
        ".validate": "$u.matches(/^[a-z0-9_]{3,20}$/) && newData.hasChildren(['s','h','c']) && newData.child('h').val().length === 44 && newData.child('s').val().length === 24"
      }
    },
    "apps_water": {
      "$u": {
        ".read": true,
        ".write": true,
        "goal": { ".validate": "newData.isNumber() && newData.val() >= 500 && newData.val() <= 6000" },
        "days": {
          "$d": { ".validate": "$d.matches(/^\\d{4}-\\d{2}-\\d{2}$/) && newData.isNumber() && newData.val() >= 0 && newData.val() <= 20000" }
        },
        "$other": { ".validate": false }
      }
    }
  }
}
```

## Known limitation (by design)

Without a backend that verifies the PIN server-side, RTDB rules can't tie a write to identity: anyone who knows a username could write to their data via a direct REST call. Acceptable for non-sensitive data like milliliters of water. Considered upgrade path: a Netlify Function that verifies the hash and signs an HMAC token (same pattern used in another app in the suite for cross-app auth), with RTDB rules based on custom auth.

Documenting security trade-offs explicitly, instead of hiding them, is a deliberate choice.

## Running it locally

1. Create a free Firebase project → enable Realtime Database
2. Publish the security rules above
3. In `index.html`, replace `RTDB_URL` with your project's URL
4. Open `index.html` in a browser (no server required — it's fully static)

> Note: some absolute paths (`/manifest.json`, `/lang.js`, `/main.js`, `/sw.js`, images) assume roversia.it's site structure. For a standalone deploy (e.g. GitHub Pages) these need to be adjusted or removed.

## Why this repo exists

This code is extracted from the app suite on [roversia.it/apps.html](https://roversia.it/apps.html), published as an example of:
- custom authentication without relying on an external provider
- using Firebase via REST instead of the SDK (more control, less weight)
- handling concurrent writes without a real backend

---

Andrea Roversi · Liguria, Italy · [roversia.it](https://roversia.it)
