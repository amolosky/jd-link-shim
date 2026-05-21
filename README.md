# Jewelry Depot — Meta In-App Browser Escape Shim

A single static HTML page that escapes Instagram / Facebook / TikTok in-app browsers and reopens the destination in **Safari (iOS)** or **Chrome (Android)**. This recovers the conversion lost when Meta traps users in its in-app WebView (no Apple Pay, no saved logins, no Shopify session, broken Klaviyo cookies, etc.).

**Single page.** One `index.html` handles every product, collection, and campaign — the destination is passed as `?to=...`.

---

## Live URL format

Point every Meta / TikTok ad at:

```
https://go.jewelrydepot.com/?to=https://shop.jewelrydepot.com/products/YOUR-HANDLE
```

UTMs and tracking params can go on either side; the shim forwards any extra query params it doesn't recognize.

Examples:

```
https://go.jewelrydepot.com/?to=https://shop.jewelrydepot.com/products/diamond-tennis-bracelet&utm_source=instagram&utm_campaign=spring26
https://go.jewelrydepot.com/?to=https://shop.jewelrydepot.com/collections/engagement-rings
https://go.jewelrydepot.com/  ← bare hit, sends to homepage
```

**Security:** the shim only redirects to `*.jewelrydepot.com` hosts. Anything else falls back to the homepage (prevents open-redirect abuse).

---

## How it works

1. Loads in ~50 ms.
2. Sniffs `navigator.userAgent` for `Instagram`, `FBAN/FBAV/FB_IAB`, or `TikTok/BytedanceWebview`.
3. If detected:
   - **Android** → `intent://...#Intent;package=com.android.chrome;end` (OS opens Chrome).
   - **iOS** → `x-safari-https://...` (OS pops out to Safari).
4. If not in-app → plain `location.replace(target)`.
5. Visible fallback link in case JS is disabled or the OS doesn't honor the scheme.

---

## Deploy — Cloudflare Pages (recommended, free, ~5 min)

1. **Create a GitHub repo** named `jd-link-shim` and push this folder:
   ```bash
   cd "/Users/amolosky/Documents/Jewelry Depot Code/jd-link-shim"
   git init
   git add .
   git commit -m "Initial shim"
   gh repo create jd-link-shim --public --source=. --push
   # or create manually on github.com and: git remote add origin … && git push -u origin main
   ```

2. **Cloudflare Pages → Create project → Connect to Git → pick `jd-link-shim`.**
   - Framework preset: **None**
   - Build command: *(leave empty)*
   - Build output directory: `/`
   - Deploy.

3. **Custom domain.** In the Pages project → *Custom domains* → add `go.jewelrydepot.com`.
   Cloudflare will tell you to add a CNAME — if `jewelrydepot.com` is already on Cloudflare DNS it does it for you in one click. Otherwise add this at your DNS host:
   ```
   CNAME   go   <your-project>.pages.dev
   ```

4. **Wait ~1–5 min** for SSL to issue, then visit `https://go.jewelrydepot.com/` — you should see the "Opening Jewelry Depot…" text for a split second then land on the homepage.

### Alt host options (any work — pick one)

| Host             | Cost | Notes                                       |
|------------------|------|---------------------------------------------|
| Cloudflare Pages | Free | Recommended. Fastest edge, free SSL.        |
| Netlify          | Free | Drop the folder into the dashboard, done.   |
| Vercel           | Free | Same idea.                                  |
| GitHub Pages     | Free | Works, but slower TTFB than Cloudflare.     |

---

## Testing checklist

Run these before flipping any ads:

- [ ] `https://go.jewelrydepot.com/` in desktop Chrome → lands on `shop.jewelrydepot.com/`.
- [ ] `https://go.jewelrydepot.com/?to=https://shop.jewelrydepot.com/products/<any-handle>` in desktop Chrome → lands on that product.
- [ ] DM the same URL to yourself on **Instagram (iOS)** → tap the link → should pop out into Safari, not stay in the IG browser.
- [ ] Same on **Instagram (Android)** → should open Chrome.
- [ ] Same on **Facebook Messenger** both platforms.
- [ ] Bad target test: `https://go.jewelrydepot.com/?to=https://evil.example.com` → should redirect to `shop.jewelrydepot.com/` (allowlist blocks it).

---

## Meta Ads Manager setup (do this alongside the shim)

Belt-and-suspenders — Meta has a native "open in system browser" toggle on some objectives. Turn it on where you can, then use the shim URL for everything else:

1. Ads Manager → your campaign → Ad set / Ad level.
2. Look for **Destination** → **Browser** → set to **Default mobile browser** (not "In-app browser") if exposed.
3. Replace the ad's destination URL with the shim URL above.

---

## Maintenance

- **Almost zero.** This page changes maybe twice a year.
- If you want stricter UTM scrubbing (to fix the Sirv / cache-bust issue separately documented in `shopify-utm-debug`), uncomment the `replace(...)` block inside `index.html` and add the params you want to strip.
- If you add a new Meta-owned in-app browser (Threads, WhatsApp), add its UA token to the `isInApp` check.

---

## Why this matters (the number)

Internal estimate: moving Meta traffic out of the in-app browser raises Instagram conversion **5–15×** and Facebook **3–8×**. At current Meta session volume (~14,783/mo flagged as torched), the shim is one of the highest-ROI single deploys available — no theme changes, no Shopify risk, no ongoing cost.
