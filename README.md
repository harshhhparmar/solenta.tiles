# Solenta Tiles — Website

Single-page static site for Solenta Tiles, a tile trading firm based in Morbi, Gujarat.

## Structure

Everything (HTML, CSS, JS, and the logo) lives in one self-contained file:

```
index.html
```

No build step, no dependencies, no framework — just a static HTML file.

## Contact form

The enquiry form on the site sends details via:
- **WhatsApp** → opens a chat to `+91 94084 80458` with the enquiry pre-filled
- **Email** → opens the visitor's email client addressed to `solentatiles1@gmail.com`

To change either, edit the `WHATSAPP_NUMBER` and `EMAIL_ADDRESS` constants near the bottom of `index.html`.

## Deploying on Vercel

1. Push this repo to GitHub (see below).
2. Go to [vercel.com/new](https://vercel.com/new) and import the `solenta.tiles` repository.
3. Framework preset: **Other** (no build command needed — Vercel will serve `index.html` as-is).
4. Click **Deploy**.

Any future push to the `main` branch will auto-redeploy.
