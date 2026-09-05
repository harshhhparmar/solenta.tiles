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

-- Only a signed-in admin account can read enquiries
-- (safe to use "true" here because only one admin account can ever
-- exist — see the "Admin panel" section below)
create policy "Admin can read all enquiries"
  on public.enquiries
  for select
  to authenticated
  using (true);
```

(If you already created the table from an earlier version of this README, just run the `alter table ... add column if not exists email text` line and the two `create policy` statements — skip re-creating the table. If you previously ran a version of this policy with a hardcoded email address, drop it first with `drop policy "Admin can read all enquiries" on public.enquiries;` before re-creating it.)

To change the Supabase project or key, edit `SUPABASE_URL` and `SUPABASE_ANON_KEY` near the bottom of `index.html`, `admin-login.html`, and `admin-dashboard.html`.

## Admin panel

There is no built-in username/password anywhere in this project. The very first person to open `admin-login.html` sets the one admin email and password themselves — nothing is pre-filled, and no credentials live in the code.

**One-time setup — run this once in your Supabase project's SQL Editor** (in addition to the `enquiries` table SQL above):

```sql
create table if not exists public.admin_setup (
  id int primary key,
  is_configured boolean not null default false,
  constraint admin_setup_singleton check (id = 1)
);

insert into public.admin_setup (id, is_configured)
values (1, false)
on conflict (id) do nothing;

alter table public.admin_setup enable row level security;

-- Anyone can check whether the admin account has been created yet
create policy "Public can read setup status"
  on public.admin_setup
  for select
  to anon, authenticated
  using (true);

-- The flag can only ever flip from false to true, and only once —
-- this is what stops a second admin account from ever being set up
create policy "Flip setup flag once, never again"
  on public.admin_setup
  for update
  to anon, authenticated
  using (is_configured = false)
  with check (is_configured = true);
```

Then, in **Authentication → Providers → Email** (or **Auth Settings**), leave **"Allow new users to sign up"** turned **ON** for now — the first-time setup screen needs it. You'll turn it off again in step 4 below.

**First-time setup, done on the live site itself:**

1. Open `yoursite.com/admin-login.html`. Since no admin account exists yet, it shows a **"Create Admin Account"** form instead of a login form.
2. Enter whatever email and password you want the admin account to use, and submit.
3. Depending on your Supabase project's email settings, you'll either be signed in immediately, or asked to click a confirmation link sent to that email before you can sign in.
4. Once you're in, go back to **Authentication → Providers → Email** in Supabase and **turn "Allow new users to sign up" back off**. This is what makes "only one admin account" permanent — without it, someone could still call Supabase's sign-up API directly (bypassing the site's UI) and create a second account.

After that one-time setup, `admin-login.html` will only ever show the plain sign-in form again — the "Create Admin Account" form is gated behind the `admin_setup.is_configured` flag, which the database itself refuses to flip back to false.

If you'd earlier created a `solentatiles1@gmail.com` admin user manually while testing a previous version of this project, delete it from **Authentication → Users** before doing first-time setup, so you end up with exactly one account, chosen by you.

**Troubleshooting — "it's asking for a password instead of letting me create an account":**

This means `admin-login.html` thinks setup is already done, almost always because the `admin_setup` table SQL above hasn't been run in your Supabase project yet (in that case the page now shows a small error note explaining that, right on the create-account form itself — if you're not seeing that note, you're on an older copy of the file). To check directly: in Supabase, **Table Editor → admin_setup**, look at the `is_configured` row for `id = 1`. If it's already `true` and you want to redo setup, run:

```sql
update public.admin_setup set is_configured = false where id = 1;
```

then also delete whatever auth user got created from **Authentication → Users**, and reload `admin-login.html`.

**Troubleshooting — "I created the account but still can't log in":**

Check **Authentication → Users** in Supabase first. If you see **more than one user** there, or a user with an old test email, that's almost certainly the problem. Supabase deliberately does not error when you "create an account" with an email that's already registered — it silently keeps the existing account and its existing password, and just shows a "check your email" message either way, so it looks like it worked. The page now detects this specific case and shows a clear warning when it happens ("This email already has an account…") — if you saw that message, sign in with whatever password that account was originally created with, or delete it from **Authentication → Users** and start over with a clean email.

Other things to check if login still fails:
- **"Email not confirmed"** — if your Supabase project requires email confirmation and the confirmation email never arrived (spam folder, or email sending not configured), go to **Authentication → Users**, open the account, and confirm it manually, or resend the confirmation from there.
- **"Signups not allowed for this instance"** while creating the account — go to **Authentication → Providers → Email** and make sure "Allow new users to sign up" is turned **on** for setup, then off again afterward.
- Check **Authentication → Logs** in Supabase for the exact rejected request — it will show precisely which check failed.
- Confirm you're testing on the deployed site (not a stale local copy) so you're running the latest `admin-login.html`.

**How it works:**
- `admin-login.html` — checks the `admin_setup` flag first. If nobody has set up an account yet, it shows the create-account form (calls Supabase Auth `signUp`, then flips the flag — unless it detects the email-already-exists case above, in which case it does not flip the flag). Otherwise it shows a plain email/password sign-in form (`signInWithPassword`). No password is ever written into the page's code — both forms send whatever the person types straight to Supabase over HTTPS.
- `admin-dashboard.html` — on load, checks for a valid Supabase session; if there isn't one, it redirects to the login page. It then queries the `enquiries` table and lists every inquiry (date/time, name, phone, email, company, product range, message, source) with a **Log out** button that ends the session.
- Real protection happens at the database, not just in this page's JavaScript: the `enquiries` table's Row Level Security only allows `select` to a signed-in Supabase user at all — and thanks to the one-time signup flag, there can only ever be one such user. Even if someone bypassed the dashboard page entirely, they could not read the data without a valid login.
- The "Admin Login" link lives only in the footer of `index.html`, styled as a small muted utility link — it does not appear in the header/navbar, and no other part of the existing site was changed.

## Deploying on Vercel

1. Push this repo to GitHub (see below).
2. Go to [vercel.com/new](https://vercel.com/new) and import the `solenta.tiles` repository.
3. Framework preset: **Other** (no build command needed — Vercel will serve the HTML files as-is).
4. Click **Deploy**.

Any future push to the `main` branch will auto-redeploy.
