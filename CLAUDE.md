# CLAUDE.md

Instructions for Claude Code (or any AI coding agent) working on Netty Pal.

Read **`SPEC.md`** first: it is the agreed specification, data model and build order.

## Who you're working with

The owner is not a professional developer. Explain technical decisions, trade-offs and
risks in plain English, with a one-line explanation of any jargon.

## Architecture

- Static site at **https://nettypal.au**, hosted on Vercel (free Hobby plan), which publishes
  `main` automatically: `index.html` (all UI and logic), `sw.js`, `manifest.json`, icons.
  No build step, no framework, no bundler. Pull requests get Vercel preview links, but
  sign-in only works on addresses listed in Firebase > Authentication > Authorised domains.
- Moved from GitHub Pages (`patrycka.github.io/netty-pal`) in October 2026. A script at the
  top of `index.html` forwards that old address to nettypal.au; keep it.
- Domain `nettypal.au` is registered at Porkbun (DNS there). Firebase sign-in emails are
  sent from `noreply@nettypal.au`; the DNS records for that (two `firebase` CNAMEs, the
  SPF and `firebase=` TXT records, `_dmarc`) must not be removed.
- Firebase (project `netty-pal`): Google sign-in and Cloud Firestore, loaded from Google's CDN.
- `firestore.rules` is the source of truth for permissions. It is **not** deployed
  automatically: after changing it, the owner must paste it into Firebase console >
  Firestore Database > Rules and click Publish. Always tell them when this is needed.
- Any new permission must be enforced in `firestore.rules`, not just hidden in the UI.
- Bump `CACHE` in `sw.js` when shipping changes so installed copies update.

## Conventions

- Match Packing Pal's style (`../Packing Pal/web/index.html`): plain JS, `render()` builds
  HTML strings, delegated `data-action` click handlers, `escapeHtml` on all user text.
- No em dashes in copy or comments.
- Never commit passwords, tokens or private keys. (The Firebase web config is not secret.)

## Git workflow

- Work on a branch and open a pull request. The owner merges; merging publishes the site.
- Never push to `main`, force-push, or rewrite history without explicit permission.

## Testing and reporting

- Serve locally (localhost is an authorised sign-in domain by default) and check the
  browser console. Validate any JSON you edit.
- Report what you tested, what you couldn't test and why, and anything to check before merging.
