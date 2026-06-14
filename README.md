# halal-meetups-links

Tiny **public** static host whose only job is to serve the Apple App Site
Association (AASA) for Halal Meetups iOS Universal Links:

```
https://links.halalmeetups.com/.well-known/apple-app-site-association
```

## Why this exists

iOS Universal Links require the AASA to be fetchable **anonymously** over HTTPS
(Apple's servers fetch it with no cookies). The dev web app
(`dev.halalmeetups.com`) is intentionally kept behind Vercel Authentication so
its **dev database** (test events/groups/members) is never publicly browsable —
which means Apple can't read an AASA hosted there.

This project sidesteps that: it contains **only** the static AASA (plus this
readme and a placeholder page) — no Supabase, no app, nothing browsable. So it
can be fully public without exposing any data.

The sign-up confirmation email (sent by the **dev** Supabase project) links to
`https://links.halalmeetups.com/auth/confirm?token_hash=…&type=signup`. When the
app is installed, iOS opens the app directly (this site is never loaded) and the
app verifies the token against dev Supabase. When the app is **not** installed,
`/auth/confirm` here just 404s — an acceptable fallback for the dev phase.

At launch, when iOS targets prod, the AASA on `www.halalmeetups.com` takes over
and this host can be retired.

## Deploy

Vercel project, **no framework** ("Other"), root directory = repo root. The
`vercel.json` sets `Content-Type: application/json` on the AASA and a blanket
`X-Robots-Tag: noindex` so search engines ignore it. Custom domain:
`links.halalmeetups.com`.
