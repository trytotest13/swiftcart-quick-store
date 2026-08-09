# SwiftCart — Vercel deployment

This is the same real-auth / real-order / Razorpay-test-mode SwiftCart backend from
`DELIVERY_NOTES.md`, restructured for a one-repo Vercel deployment:

```
api/index.js      ← the whole Express app, exported for Vercel's Node.js serverless runtime
config/db.js       ← MongoDB connection, cached across warm invocations (important on serverless)
models/, middleware/, routes/, utils/   ← unchanged backend logic
public/index.html  ← the storefront (was swiftcart-premium.html)
public/admin.html  ← the admin panel
vercel.json        ← rewrites every /api/* request to api/index.js
```

Vercel auto-serves everything under `public/` as static files and auto-detects `api/index.js`
as a serverless function — no build step required.

## Before you deploy

1. **A reachable MongoDB.** A local `mongodb://127.0.0.1...` URI will NOT work on Vercel —
   serverless functions can't reach your laptop. Create a free MongoDB Atlas cluster
   (M0 tier) and get its connection string.
2. **Razorpay TEST keys** from the [Razorpay dashboard](https://dashboard.razorpay.com/app/keys)
   (toggle to Test Mode).

## Deploy

```bash
npm i -g vercel      # if you don't already have it
cd swiftcart          # this folder
vercel                # first deploy — follow the prompts, link/create a project
```

Then in the Vercel dashboard → your project → **Settings → Environment Variables**, add:

| Variable | Value |
|---|---|
| `MONGODB_URI` | your Atlas connection string |
| `SESSION_SECRET` | `node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"` |
| `PAYMENT_KEY_ID` | Razorpay test key id (`rzp_test_...`) |
| `PAYMENT_KEY_SECRET` | Razorpay test key secret |
| `PAYMENT_WEBHOOK_SECRET` | from Razorpay Dashboard → Webhooks (point it at `https://<your-app>.vercel.app/api/payments/webhook`) |
| `SEED_ADMIN_EMAIL` / `SEED_ADMIN_PASSWORD` | used once, by the seed script below |
| `NODE_ENV` | `production` (Vercel sets this automatically, but doesn't hurt to set) |

Redeploy after adding env vars (`vercel --prod`), since functions only pick up new env vars on
a fresh deployment.

## Seed the database (once)

The seed script needs the *same* `MONGODB_URI` your deployed app uses. Run it locally against
that same Atlas cluster:

```bash
npm install
cp .env.example .env      # fill in MONGODB_URI, SEED_ADMIN_EMAIL, SEED_ADMIN_PASSWORD at least
npm run seed
```

This loads the product catalog, the three starter coupons, and creates your one admin account.
There is deliberately no way to create an admin account from either the storefront or the admin
panel itself — see `middleware/auth.js` and `routes/auth.js`.

## Verify

- `https://<your-app>.vercel.app/` → storefront
- `https://<your-app>.vercel.app/admin.html` → admin panel (sign in with `SEED_ADMIN_EMAIL`)
- `https://<your-app>.vercel.app/api/health` → `{"ok":true}`

Then walk through the registration/login, admin-403, and UPI test-payment checks in
`DELIVERY_NOTES.md` §9–12 against your live deployment.

## One thing worth knowing about this environment

`argon2` (used for password hashing) is a native addon. Vercel's Node.js runtime builds on
Linux/x64 and this normally works fine with prebuilt binaries, but if a deploy ever fails on
that dependency specifically, the fallback is swapping to `bcryptjs` (pure JS, no native
build) in `routes/auth.js`'s `argon2.hash`/`argon2.verify` calls and `package.json`. I haven't
hit this deploying real projects, but flagging it since I can't run a real deploy from here to
confirm.
