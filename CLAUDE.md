# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this is

**VoroCreator Studio** (`vorominiapp`) is a **Telegram Mini App** — a single-page
web app that runs inside Telegram's in-app browser (WebView). It's the front end
for **voro.uz**, an AI generation service that lets users create AI **videos** and
**images** from text prompts and reference media, pay with an in-app "tangacha"
(⚡) credit balance, and top up via Telegram Stars.

The entire app is **one file**: `index.html` (~2,470 lines, ~416 KB). There is no
build step, no package manager, no framework CLI. It is deployed as a static file
via **GitHub Pages** (`.nojekyll` disables Jekyll processing so the raw file is
served as-is).

The UI is primarily in **Uzbek** (`<html lang="uz">`), with full runtime
translations for **Russian** and **English** (see i18n below). Code comments are
mostly Uzbek — keep new comments consistent with the surrounding style.

## Architecture at a glance

- **React 18** loaded from CDN as UMD globals (`react.production.min.js`,
  `react-dom.production.min.js`) — no imports, `React` and `ReactDOM` are globals.
- **Babel Standalone** (`@babel/standalone`) transpiles the JSX **in the browser
  at runtime**. The entire app lives in one `<script type="text/babel">` block
  (starts ~line 19). This is why there's no build tooling — the browser compiles it.
- **Telegram Web App SDK** (`telegram-web-app.js`) provides `window.Telegram.WebApp`
  (haptics, BackButton, CloudStorage, theme colors, `initData`, `sendData`).
- **Icons** are hand-inlined Lucide SVGs via the `_ic(paths)` factory (~line 24+) —
  no icon library dependency.
- **Styling** is inline JS style objects plus one `<style>` block. Design tokens
  live in `c` (colors), `glass`, `glossy`, `TONES`, `font`, `noise` (~line 774+).
  The aesthetic is a dark, glassmorphic, iOS-like theme (`#070809` background).

Because everything is global and single-file, there are **no modules or exports** —
components and constants are top-level `function`/`const` declarations in one scope.

## Backend / API

The app is a thin client. All real work happens on the backend:

```js
const API_BASE = 'https://voroapi.duckdns.org';   // ~line 754
```

### Auth
Every request carries Telegram identity via headers, built in `apiFetch` /
`apiPost` (and duplicated in upload/enhance calls):
- `X-Tg-Init-Data` — Telegram `initData` (the signed launch payload), when present.
- `X-Tg-Uid` / `X-Tg-Sig` / `X-Tg-Name` — a bot-signed `uid`+`sig`(+`n`) taken from
  the **URL query string** (`?uid=&sig=&n=&bal=`). This is the fallback path used
  when the Mini App is opened from a **KeyboardButton**, where `initData` arrives
  empty. `URLQ` (~line 756) parses these.
- `X-Lang` — current UI language.

### Endpoints used
| Method | Path | Purpose |
|--------|------|---------|
| GET  | `/custom-models` | Dynamically-added models merged at runtime by `applyCustomModels` |
| GET  | `/tg/me` | Live user name + balance (triggers lazy account migration server-side) |
| GET  | `/tg/templates` | Home-screen templates |
| GET  | `/tg/tools` | Photo tools |
| GET  | `/tg/history` | Gallery items (`GalleryScreen`) |
| POST | `/tg/enhance` | AI prompt-enhancement (vision-aware; body: `{prompt, mid, res, asp, dur, refs:[id]}`) |
| POST | `/tg/upload` | Multipart file upload (`FormData` field `file`) → returns a ref `{id, url}` |
| POST | `/tg/generate` | Start a job → `{job_id, balance}` (HTTP **402** = insufficient balance) |
| GET  | `/tg/job/:id` | Poll job status → `{status, progress, result_url, balance, error}` |

### Generation flow (the core loop)
1. User picks a model → `ParamsScreen` collects prompt, resolution, aspect,
   duration, and uploaded refs.
2. `ConfirmSheet` shows the computed price and balance check.
3. `confirmSend` POSTs `/tg/generate` with `{mid, res, asp, dur?, prompt?, refs?}`.
4. The app sets a local `job` state and **polls `/tg/job/:id` every ~3s** until
   `status === 'done'` (shows `result_url`) or `'failed'`.
5. `ResultSheet` displays the result; media is served from `API_BASE + result_url`.

Templates and tools use the same `/tg/generate` + poll flow with different payloads
(`{template_id, style_id, refs}` and `{tool_id, refs, fields}` respectively) — see
`templateGenerate` and `toolGenerate` in `App`.

## Key data structures (top of the script)

- `VIDEO_MODELS` / `IMAGE_MODELS` (~line 80, 107) — the model catalog. Each entry:
  `{ id, name, emoji, badge, meta: {...} }`. `id` is the backend model slug
  (e.g. `google/veo3.1/reference-to-video`) and is the source of truth everywhere.
  `meta` declares `resolutions`, `aspects`, `dur_steps`, `refs` (max ref count),
  `refs_req`, and capability flags: `needImage`, `needVideo`, `needAudio`,
  `imgReq`, `vidReq`, `audReq`, plus localized `hook`/`hookRu`/`hookEn` blurbs.
  Note these are `let` — `applyCustomModels` mutates them at runtime.
- `DEFAULT_META` (~line 78) — merged under every model via `getMeta`.
- `PRICING` (~line 141) — price in ⚡ keyed by model `id`. Shapes: `{dur:{...}}`,
  `{res:{...}}`, `{res_dur:{...}}`, or `{perSec, unit, max}` for duration-billed
  models (video/audio-edit). **Must stay in sync with the bot's
  `calc_video_tangacha`/`calc_image_tangacha`** — the comment notes the sync date.
  Read prices via `getPrice`, `perSecInfo`, `minPriceText` — don't index directly.
- `STAR_PACKAGES` (~line 121) — top-up bundles priced in Telegram Stars.
- `FEATURED_IDS` (~line 128) — model ids featured on the home screen.
- `applyCustomModels(list)` (~line 2168) — merges `/custom-models` results,
  tagging them `_custom: true` (removed and re-added on each apply) and registering
  their pricing. This lets the backend add models without shipping a new `index.html`.

## Internationalization (i18n)

- `I18N` (~line 203) is `{ uz: {...}, en: {...}, ru: {...} }` — flat key→string maps.
- `CUR_LANG` is a **module-level mutable global** (not React state). `t(key, fallback)`
  (~line 737) reads from `I18N[CUR_LANG]`, falling back to the passed default
  (usually the Uzbek string) then the key itself.
- Language is resolved (`detectLang`) from Telegram `CloudStorage` (`voro_lang`) →
  Telegram user `language_code` → `uz`. `setLang` writes back to `CloudStorage` and
  bumps React state so the tree re-renders. Model `hook` text is localized inline
  via the `hook`/`hookRu`/`hookEn` fields rather than through `t()`.

When adding user-facing text: add the key to **all three** language maps in `I18N`
and read it through `t('key', 'uzbek default')`.

## Component map (all in `index.html`)

- `App` (~line 2195) — root: owns all top-level state (tab, model, job, balance,
  templates, tools, lang), the Telegram lifecycle `useEffect`s (ready/expand/theme,
  BackButton routing), and the generate/poll orchestration.
- Screens: `HomeScreen`, `ModelListScreen`, `ParamsScreen`, `TemplateScreen`,
  `ToolListScreen`, `ToolScreen`, `GalleryScreen`, `ProfileScreen`, `BuyScreen`.
- Sheets/overlays: `ConfirmSheet`, `ResultSheet`, `InfoSheet`, language sheet
  (inline in `App`).
- Atoms: `IconTile`, `EmojiTile`, `VLogo`, `Avatar`, `Header`, `Label`, `Chips`,
  `Segmented`, `DurationSlider`, `Group`, `Row`, `TabBar`, `Spinner`.
- "Routing" is plain conditional rendering on the `tab` string plus which
  `model`/`activeTemplate`/`activeTool`/`job` is set (see the return block
  ~line 2431). The Telegram **BackButton** handler (~line 2264) mirrors this stack.

## Telegram integration helpers (~line 189, 770)

- `TG()` — safe accessor for `window.Telegram.WebApp` (returns `null` outside TG).
- `send(obj)` — `tg.sendData(JSON.stringify(obj))`; outside Telegram it `alert`s the
  payload (dev fallback).
- `haptic(t)` / `hapticOk()` — haptic feedback wrappers, all guarded in try/catch.
- All Telegram calls are defensively wrapped so the app still renders in a plain
  browser (useful for local preview).

## Development workflow

There is no install/build/test tooling. To work on this app:

- **Edit** `index.html` directly. Keep everything inside the single
  `<script type="text/babel">` block; new components/constants go at top level.
- **Preview locally** by serving the folder over HTTP (e.g. `python3 -m http.server`)
  and opening `index.html`. Outside Telegram, `TG()` returns `null`, balance/name
  come from URL params or defaults, and `send`/upload paths degrade gracefully — but
  real generation needs the live backend and Telegram auth.
- **No linter/formatter/test suite** is configured. Match the existing terse,
  single-line-dense style and inline-style conventions.
- Because Babel transpiles in-browser, a **syntax error breaks the whole app
  silently** (blank screen) — check the browser console after edits.
- Keep the file self-contained; do not introduce a bundler, npm dependencies, or
  external module files unless explicitly asked — deployment is a raw static file.

### Deployment
Static hosting via GitHub Pages. `.nojekyll` is intentional — do not remove it.
Pushing the branch that Pages serves updates the live app; there is no CI build.

## Conventions & gotchas

- **Model `id` is the contract** with the backend and appears in `PRICING`,
  `FEATURED_IDS`, `isVideoModel`, capability checks, and payloads. Never rename an
  `id` without coordinating the backend.
- **Pricing must match the bot.** The frontend price is display/UX only; the server
  is authoritative and deducts on `/tg/generate`. Still keep `PRICING` accurate to
  avoid confusing users, and note the sync date in the comment when you change it.
- **Add a model in three places** if adding a static one: the `VIDEO_MODELS` or
  `IMAGE_MODELS` array **and** `PRICING`. Prefer backend `/custom-models` for
  additions that shouldn't require a redeploy.
- **i18n:** every visible string goes through `t()` and exists in `uz`/`en`/`ru`.
- **Defensive Telegram access:** always guard `window.Telegram` access in try/catch
  or via `TG()`; the app must not throw when Telegram APIs are missing.
- **Uploads:** use `FormData` field name `file`; the upload returns a ref object
  whose `id` is what generation payloads reference (`refs: [id, ...]`), while
  `url` (prefixed with `API_BASE`) is used for local previews.
- Comments and many identifiers are Uzbek (`tangacha` = credits, `narx` = price,
  `shablon` = template, `vosita` = tool, `ijodkor` = creator). Keep them consistent.

## Git & branches

- Active development branch for this work: `claude/claude-md-docs-n5e903`.
  Default branch: `main`. Develop on the designated branch, commit with clear
  messages, and push there — do not push elsewhere without permission.
- Commit history messages are short and often Uzbek (e.g. "Video edit nisbat",
  "Custom modellar avtomatik") — brief, descriptive summaries are the norm.
