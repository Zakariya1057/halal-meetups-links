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
`/auth/confirm` shows a small "open this on your iPhone" page (the token can only
be verified inside the app, so there's nothing useful to do on the web).

At launch, when iOS targets prod, the AASA on `www.halalmeetups.com` takes over
and this host can be retired.

## Web fallback (app not installed)

Every link in the **dev** transactional emails points at this host
(`NEXT_PUBLIC_EMAIL_LINK_URL=https://links.halalmeetups.com`) so iOS Universal
Links can open the app. When the app **is** installed, iOS intercepts the tap and
this host is never hit. When it **isn't** installed (desktop, Android, or an
iPhone without the app), the browser loads the URL here — and without a fallback
every path 404s.

`vercel.json` fixes that with a single catch-all redirect: every path **except**
the AASA and `/auth/confirm` is 307-redirected to the matching path on the dev
web app, preserving the path **and** query string:

```
https://links.halalmeetups.com/events/abc?x=1
  → 307 → https://dev.halalmeetups.com/events/abc?x=1
```

This covers all paths the dev emails route here (`/events`, `/groups`, `/invite`,
`/member/*`, `/billing/pay`, `/pricing`, `/sign-up`, `/home`) with no per-path
upkeep — a new path prefix is caught automatically.

**Target = dev, on purpose.** Dev links reference dev data (dev slugs, dev
tokens) that only exist on `dev.halalmeetups.com`, so that's where they must
land. `dev.halalmeetups.com` sits behind Vercel Authentication; the people
clicking dev links are authenticated testers, so the login wall is expected and
keeps the test database private. (`www.halalmeetups.com` is wrong here — it's
public but has none of the dev rows, so every deep link would 404.)

**To switch dev → live** (e.g. if this host is ever pointed at prod instead of
retired), change the one `destination` origin in `vercel.json` from
`https://dev.halalmeetups.com` to `https://www.halalmeetups.com` and redeploy.

The redirect is **307 (temporary)** on purpose — this host is short-lived and the
target may change, so browsers must not cache it permanently.

## Deploy

Vercel project, **no framework** ("Other"), root directory = repo root.
`vercel.json` sets `Content-Type: application/json` on the AASA, a blanket
`X-Robots-Tag: noindex` so search engines ignore it, and the web-fallback
redirect above. Custom domain: `links.halalmeetups.com`. Pushing to `master`
auto-deploys the live host — verify the AASA still serves (see below) after any
`vercel.json` change.

## Verify after deploy

```sh
# AASA must still be a direct 200 with JSON content-type (NOT redirected) —
# a redirect here breaks Universal Links for the app.
curl -sI https://links.halalmeetups.com/.well-known/apple-app-site-association

# A content path should 307 to dev, preserving path + query.
curl -sI "https://links.halalmeetups.com/events/x?foo=1"   # → location: https://dev.halalmeetups.com/events/x?foo=1

# /auth/confirm should still serve the friendly page (200), not redirect.
curl -sI https://links.halalmeetups.com/auth/confirm
```
