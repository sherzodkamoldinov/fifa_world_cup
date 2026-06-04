# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file static web app: a **FIFA World Cup 2026 predictor** (UI text in Uzbek) plus a real-time **chat** with Google login and user profiles. Everything — HTML, CSS, all JavaScript — lives in `index.html` (~2000 lines). There is no build system, no package.json, no tests, no dependencies installed locally. Firebase is loaded from a CDN at runtime.

## Commands

- **Run/preview:** open `index.html` in a browser (`open index.html` on macOS). Note: Google sign-in popups do **not** work from the `file://` protocol — auth/chat must be tested on the deployed site or a local http server (`python3 -m http.server`).
- **No build, no lint, no test runner.** To sanity-check JS before committing, extract a `<script>` block and run `node --check`. The classic script and the `type="module"` script must be checked separately (the module uses ESM `import`):
  ```bash
  python3 -c "import re;h=open('index.html').read();open('/tmp/m.js','w').write(re.search(r'<script>(.*?)</script>',h,re.S).group(1))" && node --check /tmp/m.js
  ```
- **Deploy:** push to `main`. GitHub Pages auto-deploys to `https://sherzodkamoldinov.github.io/fifa_world_cup/`. The entry file **must stay named `index.html`** for the root URL to work. Allow ~1 min after push, then hard-refresh (Cmd+Shift+R) — browser caching frequently serves a stale version.
- This is a static-host project; GitHub Pages requires the repo to stay **public** on the free plan.

## Two-script architecture (important)

`index.html` has **two** script blocks that share state via the `window` object:

1. **Classic `<script>`** (the bulk): all app logic — tournament data, standings, knockout bracket, chat UI rendering, profile UI. Runs during parse.
2. **`<script type="module">`** at the end: initializes Firebase (App, Realtime Database, Google Auth) from the gstatic ESM CDN and exposes `window.fbChat` and `window.fbAuth`. Because module scripts are deferred, it runs *after* the classic script. The classic script registers a `window.addEventListener('fbchat-ready', ...)` listener; the module dispatches that event once `window.fbChat` is ready. If Firebase init throws (offline, etc.), the chat silently falls back to a `localStorage` demo mode (`chatMode` stays `'local'`).

When changing Firebase calls, edit the **module** block; when changing how the app *uses* them, edit the classic block. The `firebaseConfig` (incl. `databaseURL`, region `asia-southeast1`) lives in the module block. Firebase keys are intentionally public — security is enforced by database rules, not secrecy.

## Tournament data model (classic script, near top)

- `GROUPS` — 12 groups A–L, each `{teams:[{n,f}]}` (name + flag emoji).
- `GM` — group matches keyed by group letter, each `{h,a,dt,v}` (home/away names, date, venue).
- `R32/R16/QF/SF/BRONZE/FINAL` — knockout rounds; team slots are encoded labels (e.g. `A1`, `W(sf_101)`, `3rd-A/B/C/D`) resolved at render time.
- User-entered scores live in `localStorage` under `state.scores`. **Group** score key = `` `${gk}_${m.h}_${m.a}` ``; **knockout** key = the match `id`.

**Critical invariant:** a team's name string must be **byte-identical** in `GROUPS` and `GM` (and any label that references it), because standings are matched by exact name (`teams.find(t => t.n === m.h)`). A mismatch silently drops the match from the table.

**Apostrophe gotcha (real past bug):** several names contain a `'` (e.g. `O'zbekiston`). Score keys therefore contain apostrophes. **Never interpolate a score key into an inline event handler** like `oninput="setScore('${sk}',...)"` — the apostrophe terminates the JS string and the handler silently throws, so scores appear typed but never save. Group score inputs use `data-key`/`data-side` attributes plus a single delegated `input` listener on `#groups-container` instead. Keep it that way for anything that passes names/keys to handlers.

## Standings rendering & the scroll/focus rule

`setScore` must **not** rebuild the whole groups container on each keystroke (that collapses page height → scroll jumps to top and inputs lose focus). Instead it computes the affected group from `key.split('_')[0]` and calls `updateStandings(gk)`, which rewrites only that group's `<tbody id="standings-${gk}">` via the shared `standingsRows(gk)` helper. Preserve this pattern when touching score input or standings code.

## Chat / profiles (Firebase Realtime Database)

Data is **normalized by uid**:
- `/users/{uid}` = `{name, photo, ts, favTeams?}` — written via `upsertUser` using `update()` (partial merge, so login doesn't wipe profile fields). Profile is upserted on login.
- `/messages/{pushId}` = `{uid, text, ts, replyTo?:{uid,text}}` — messages store **only** the uid; name/photo are resolved at render time from `usersMap` (populated by `onUsers`). `authorOf(m)` does the lookup with a fallback to any legacy embedded `name`/`photo` on old messages.

Render conventions in `renderChat`: consecutive messages from the same uid are grouped (name/avatar shown only on the first); own messages are right-aligned gold with name/avatar hidden; others use a single color (blue) — there is no per-user color. Reply context, fav-team flags under names, and clickable avatars/names (open `openUserProfile` modal) all key off uid.

**Security rules are NOT in this repo** — they live in the Firebase console (Realtime Database → Rules). If you change the message or user schema (e.g. required fields), the rules' `.validate`/`hasChildren` clauses must be updated in the console or writes will fail with `permission-denied`. Read is public; writing messages requires `auth != null` and `uid === auth.uid`.

## Workflow

Commit and push after each change; GitHub Pages is the deploy. Keep all code in `index.html` — do not split into separate files (it would break the single-file static deploy assumption).
