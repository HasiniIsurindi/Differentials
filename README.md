# Differential — backend

Real accounts (signup/login), password hashing, sessions, and a Doctor's
Dashboard where saved cases can be reviewed later for learning. Built with
Next.js API routes + Prisma + SQLite (local) / Postgres (production).

## What's in here

```
app/
  layout.js, page.js        — renders the app
  api/
    auth/signup/route.js    — POST { name, email, password, role } → creates account, signs in
    auth/login/route.js     — POST { email, password } → signs in
    auth/logout/route.js    — POST → clears session
    auth/me/route.js        — GET  → returns current user (session restore on page load)
    cases/route.js          — GET (list your saved cases) · POST (save a new case)
    cases/[id]/route.js     — GET (full case detail) · DELETE
lib/
  db.js                      — Prisma client singleton
  auth.js                    — password hashing, JWT sessions, getCurrentUser()
prisma/
  schema.prisma              — User and Case models
components/
  DxAssistant.jsx            — the whole frontend (login/signup, domains, assessment,
                                results, Save Case, Doctor's Dashboard, case detail)
```

## How auth works

- Passwords are hashed with bcrypt before they ever touch the database — the
  plain password is never stored.
- On signup/login, a JWT is signed and set as an **httpOnly cookie**
  (`dx_token`) — not accessible to JS in the browser, which is the standard
  defence against XSS stealing sessions.
- Every protected API route (`/api/cases/*`) calls `getCurrentUser()`, which
  reads that cookie, verifies it, and looks the user up — requests without a
  valid session get a 401.
- On app load, the frontend calls `/api/auth/me` once to restore the session
  if the cookie is already there (so refreshing the page doesn't log you out).

## How Save Case / Dashboard works

- After running an assessment, the Results screen has a **"Save case"**
  button. It POSTs the patient details, selected findings, vitals, and the
  computed differentials to `/api/cases`, tied to your logged-in account.
- **Doctor's Dashboard** (the icon in the top bar) lists every case you've
  saved, most recent first. Clicking one loads the full saved summary —
  patient info, the findings that were recorded, and the ranked differentials
  with their supporting evidence, exactly as they appeared at the time.
- This is deliberately framed as a *learning* tool: a growing, searchable
  library of real cases you've worked through, not just a live diagnostic aid.

## Running it locally

> **Note:** `package.json` pins Prisma to `6.19.3`. Prisma 7 changed how
> database connections are configured (the `url` field inside
> `schema.prisma` is no longer supported — it now wants a separate
> `prisma.config.ts` file and a driver adapter), which breaks this classic,
> widely-documented setup. Pinning to v6 keeps things simple; there's no need
> to move to v7 for a project this size.

```bash
npm install
cp .env.example .env
```

Open `.env` and set `JWT_SECRET` to a real random string:
```bash
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
```
Paste the output in as `JWT_SECRET`.

Then create the database and tables:
```bash
npx prisma migrate dev --name init
```
(This also runs `prisma generate`, which builds the actual database client —
`npm install` does this automatically too via `postinstall`, so most of the
time you won't need to run it by hand.)

Start the app:
```bash
npm run dev
```
Open `http://localhost:3000` — you should land on a Sign in / Create account screen.

## Merging into your existing project

If you already have the `clinical-dx-assistant` Next.js project from earlier:
1. Copy `prisma/`, `lib/`, and `app/api/` into your existing project (same paths).
2. Replace your `components/DxAssistant.jsx` with the one in this package.
3. Add the new dependencies:
   ```bash
   npm install @prisma/client bcryptjs jsonwebtoken prisma
   ```
4. Add the `postinstall`/`build` script changes from this `package.json`.
5. Follow "Running it locally" above.

## Deploying (Render / Vercel)

SQLite is great for local dev but doesn't reliably persist on most hosting
platforms (the filesystem gets reset on redeploy). For production, switch to
a hosted Postgres — a free tier from **Neon** or **Supabase** works well and
takes two changes:

1. In `prisma/schema.prisma`, change:
   ```prisma
   datasource db {
     provider = "postgresql"   // was "sqlite"
     url      = env("DATABASE_URL")
   }
   ```
2. Set `DATABASE_URL` in your hosting platform's environment variables to the
   Postgres connection string Neon/Supabase gives you (and set `JWT_SECRET`
   there too — never commit `.env`).

Then, before your first deploy, run once from your machine (pointed at the
production database):
```bash
npx prisma migrate deploy
```

The `build` script already runs `prisma generate` automatically on every
deploy, so no extra build-command changes are needed on Render/Vercel beyond
the standard `npm install && npm run build`.

## A note on how this was tested

This project was written and compiled with `next build` in a sandboxed
environment before being handed to you — the frontend, all API routes, and
the overall app compiled successfully. The one thing that *couldn't* be
verified in that sandbox is the Prisma client generation step itself, because
that sandbox's network policy blocks Prisma's binary download servers. That's
a restriction specific to where this was built, not a problem with your
machine — `npx prisma generate` / `npx prisma migrate dev` are completely
standard commands that will work normally for you.
