# Solenta Tiles — Website

Single-page static site for Solenta Tiles, a tile manufacturing firm based in Morbi, Gujarat.

## Structure

```
index.html            — the public website (unchanged UI/UX)
admin-login.html       — admin sign-in page
admin-dashboard.html   — protected dashboard listing all inquiries
```

No build step, no dependencies, no framework — just static HTML files. The admin pages are separate files, so the public site is completely untouched except for one small "Admin Login" link added to the footer (not the header/navbar).

## Contact form

The enquiry form on the site sends details via:
- **WhatsApp** → opens a chat to `+91 94084 80458` with the enquiry pre-filled
- **Email** → opens the visitor's email client addressed to `solentatiles1@gmail.com`

To change either, edit the `WHATSAPP_NUMBER` and `EMAIL_ADDRESS` constants near the bottom of `index.html`.

Every submission (via either button) is also saved to a Supabase table called `enquiries`, so you have a permanent record even if a visitor's WhatsApp/email app doesn't open correctly, and so it shows up in the **admin dashboard**. See **Supabase backend** below for one-time setup.

Note: the visible form does not currently collect a visitor email address (only name, company, phone, product range and message), since the brief was to leave the existing UI unchanged. The dashboard still has an Email column for future-proofing — it will show "—" until the form is changed to collect it.

## Supabase backend

The site is connected to Supabase project `durvxhvttqbrakvikwfk` using its public **anon/publishable key** (safe to expose in client-side code — it can only do what your Row Level Security policies allow).

**One-time setup — run this once in your Supabase project's SQL Editor:**

```sql
create table if not exists public.enquiries (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz not null default now(),
  name text not null,
  company text,
  phone text not null,
  email text,
  product_range text,
  message text,
  source text
);

alter table public.enquiries enable row level security;

-- Public website can insert new enquiries, but never read them
create policy "Allow public inserts"
  on public.enquiries
  for insert
  to anon
  with check (true);

-- Only the signed-in admin account can read enquiries
create policy "Admin can read all enquiries"
  on public.enquiries
  for select
  to authenticated
  using (auth.jwt() ->> 'email' = 'solentatiles1@gmail.com');
```

(If you already created the table from an earlier version of this README, just run the `alter table ... add column if not exists email text` line and the two `create policy` statements — skip re-creating the table.)

To change the Supabase project or key, edit `SUPABASE_URL` and `SUPABASE_ANON_KEY` near the bottom of `index.html`, `admin-login.html`, and `admin-dashboard.html`.

## Admin panel

**One-time setup — create the single admin account:**

1. In the Supabase Dashboard, go to **Authentication → Users → Add User**.
2. Email: `solentatiles1@gmail.com`
3. Password: `solenta@ap.007`
4. Check **"Auto Confirm User"** so it can sign in immediately without an email confirmation step.
5. Go to **Authentication → Providers → Email** (or **Auth Settings**, depending on your Supabase version) and **turn off "Allow new users to sign up"**. This is what actually enforces "only one admin account" — without it, someone could call the Supabase sign-up API directly and create a second account, even though this site has no sign-up page.

The password is never stored or exposed in the site's code — it lives only in Supabase's authentication system. The pages send the email/password the admin types to Supabase's login API over HTTPS; nothing is hardcoded.

**How it works:**
- `admin-login.html` — a plain email/password form that calls Supabase Auth (`signInWithPassword`). No signup form exists anywhere on the site.
- `admin-dashboard.html` — on load, checks for a valid Supabase session and that its email matches `solentatiles1@gmail.com`; if either check fails, it redirects to the login page. It then queries the `enquiries` table and lists every inquiry (date/time, name, phone, email, company, product range, message, source) with a **Log out** button that ends the session.
- Real protection happens at the database, not just in this page's JavaScript: the `enquiries` table's Row Level Security only allows `select` to a logged-in Supabase user whose email is exactly `solentatiles1@gmail.com` (see the SQL above). Even if someone bypassed the dashboard page entirely, they could not read the data without that login.
- The "Admin Login" link lives only in the footer of `index.html`, styled as a small muted utility link — it does not appear in the header/navbar and no other part of the existing site was changed.

## Deploying on Vercel

1. Push this repo to GitHub (see below).
2. Go to [vercel.com/new](https://vercel.com/new) and import the `solenta.tiles` repository.
3. Framework preset: **Other** (no build command needed — Vercel will serve the HTML files as-is).
4. Click **Deploy**.

Any future push to the `main` branch will auto-redeploy.
