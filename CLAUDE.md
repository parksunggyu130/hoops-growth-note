# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

농구선수 성장 기록 앱 ("Hoops Growth Note") — a single-page vanilla JS app for tracking a youth basketball player's growth (height/weight), diet, exercise routines, weekly schedule, and news-article scrapbook. No framework, no bundler, no TypeScript. The entire app (HTML/CSS/JS) lives in one file. It ships as an Android app via Capacitor and can also be deployed as a static PWA.

There is no git repository here (not a git repo) — do not assume `git status`/`git diff` workflows apply.

## Repo layout — three copies of the app, one source of truth

- **`index.html`** (repo root) — the actual source. All edits happen here.
- **`www/index.html`** — a build artifact, byte-for-byte copy of the root `index.html`, produced by `npm run web`. This is what Capacitor packages into the Android app. Never hand-edit this file — it will be overwritten.
- **`deploy/index.html`** — the static-web/PWA deployment copy. It is *not* auto-generated: it's `index.html` plus three extra `<link>` tags in `<head>` for `apple-touch-icon.png`, `manifest.webmanifest`, and `icon-192.png`. When changing `index.html`, if the change should also ship to the web deployment, manually re-apply it to `deploy/index.html` (or copy root over it and re-add those three lines).
- **`mockup.html`** — a standalone gallery page that iframes `index.html?mock&tab=...&open=...` at various query params to preview every screen at once with sample data (nothing is persisted in mock mode). Useful for visually reviewing UI changes without going through app flows manually.
- **`assets/`** — Android/Capacitor icon and splash source images (used by `@capacitor/assets`).
- **`deploy/`** — also holds the generated PWA icons (`icon-192.png`, `icon-512.png`, `apple-touch-icon.png`) and `manifest.webmanifest`.
- **`android/`** — the Capacitor-generated native Android project (Gradle). Treat as generated; only touch `android/app` config intentionally (e.g. signing, permissions).
- **`dist/`** — build output APK.

## Commands

```
npm run web    # copies index.html -> www/index.html
npm run sync   # runs `web`, then `npx cap sync android` (syncs www/ into the Android project)
```

There is no lint, test, or bundler step — this project has none configured. Verify changes by opening `index.html` (or `mockup.html`) directly in a browser, or via the `run` skill.

To build/install the Android app, use standard Capacitor/Gradle flow after `npm run sync` (e.g. `npx cap open android` or Gradle directly in `android/`) — there's no npm script for it.

## Architecture (all inside `index.html`)

Single `<script>` block, no modules. Key parts, top to bottom:

**State**
- `ALL` — top-level store: `{v:2, cur: <playerId>, players: {[id]: playerState}}`. Supports multiple players (children), switchable via a "선수 선택" (player picker) sheet.
- `S` — the *currently active* player's state object (a shortcut into `ALL.players[ALL.cur]`). Almost all read/write logic operates on `S` directly, then calls `save()`.
- Per-player state shape: `{v, profile, goal, goals[], growth[], meals{date:...}, routine[], routineLog{date:[routineIds]}, schedule{dow:[...]}, news[]}`.
- `save()` writes `ALL` back into `localStorage` under key `hoopsGrowth.v2` (`KEY2`). There's a legacy single-player format `hoopsGrowth.v1` (`KEY`) that `loadAll()` auto-migrates into v2 on load.
- Article-scrap screenshots are stored as `Blob`s in IndexedDB (`hoopsShots` DB, one entry per article id) via `shotPut`/`shotGet`/`shotDel` — kept out of localStorage because of size. `urlCache`/`dropUrls` manage `URL.createObjectURL` object-URL lifecycle for displaying them; always pair a shown blob URL with eventual `dropUrls`/revoke to avoid leaks.
- `?mock` query param switches persistence to an in-memory `Map` (`memShots`) instead of IndexedDB, for the mockup gallery.

**Rendering**
- No virtual DOM / diffing. `views` is a plain object of functions keyed by tab name (`home`, `growth`, `diet`, `ex`, `news`), each returning an HTML template string for `#view`.
- `render(keepScroll)` sets `#view.innerHTML` from `views[tab]()` and updates the header/tab-bar active state; pass `true` to preserve scroll position (used after in-place toggles so the whole view doesn't jump).
- Modals/forms are "sheets": `openSheet(html, full, lock)` / `closeSheet()` inject markup into a bottom-sheet `#sheet` element. Sheet-open state is pushed to browser history so Android hardware back (and the `popstate` listener) closes the sheet instead of navigating away.
- `sub` object tracks secondary in-tab toggles (e.g. `diet: 'today'|'guide'`, `ex: 'today'|'week'`).

**Event handling — declarative dispatch, not per-element listeners**
- Every interactive element carries `data-act="<name>"` (+ `data-id`/`data-v`/`data-k`/`data-d` as needed). A single delegated `click` listener on `document` switches on `el.dataset.act` to run the action (mutate `S`/`ALL`, `save()`, `render()`).
- Forms use `data-form="<name>"` the same way, handled by a single delegated `submit` listener.
- Checkbox-like meal/habit inputs use a delegated `change` listener (`[data-meal]`); the live news search box (`#q`) uses a delegated `input` listener.
- When adding a new interactive feature, follow this pattern: add `data-act`/`data-form` + a `switch` case, rather than attaching a new listener.

**URL query params drive initial mock/deep-link state** (see `init()` at the bottom): `?mock`, `?tab=`, `?sub=key:value`, `?open=register|players|profile|scrap|new`. `mockup.html` relies on these to render each screen.

**Capacitor integration** is minimal and defensive: `window.Capacitor?.Plugins?.App` is optionally used for Android hardware back-button handling (close sheet → go to home tab → exit app), wrapped in try/catch so the same `index.html` still works as a plain web page/PWA when Capacitor isn't present.

**Opening external links from JS**: the Android WebView does not implement `onCreateWindow`, so plain `window.open(url)` is silently swallowed on-device (it works fine in a desktop browser, which is easy to miss when only testing in a browser). Use the `openExternal(url)` helper instead — it fakes a real `<a target="_blank">` click, which the WebView's normal navigation interception *does* handle and correctly hands off to the system browser. A static `<a href target="_blank">` in markup (e.g. the "원문 열기" link in `scrapSheet`) is fine as-is; this only matters for links opened via JS.

**Styling** is a single `<style>` block using CSS custom properties (`--o` orange brand color, `--navy`, etc.) — no CSS framework. Layout simulates a phone frame (`#app`, max-width 480px / fixed size on wide viewports) since this is a mobile-first app even when opened in a desktop browser.

**Language**: all UI copy, comments, and data (food guides, etc.) are in Korean. Keep new user-facing strings in Korean unless told otherwise.

## Working conventions specific to this file

- This is dense, minified-style JS (short var names, chained ternaries, template literals) — match the existing terseness rather than introducing a different style for new code.
- Because everything is one file with global mutable state (`S`, `ALL`, `tab`, `sub`, `q`, `pending`, etc.), be careful about ordering: functions reference `S` assuming it's already set for the current player.
- After editing `index.html`, remember to run `npm run web` (and `npm run sync` if testing on Android) so `www/` picks up the change — Capacitor builds from `www/`, not the root file.
- If a change affects the web/PWA deployment, also reflect it in `deploy/index.html` (remember it has the extra manifest/icon `<link>` tags root doesn't).
