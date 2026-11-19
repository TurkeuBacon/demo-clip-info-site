# Demo Gallery — Technical Specification

This is a complete specification for generating a working "demo gallery" project. An LLM given this file should be able to produce the entire codebase. It was derived from a real, working implementation.

## What you are building

A video gallery site for students:

- **Home page** — a 3-column grid of video cards. Each card shows a thumbnail captured from the video at a timestamp the student chose, an optional timestamp badge, and an optional title. Clicking a card opens the watch page.
- **Watch page** — a video player (streamed through a same-origin proxy) and a comments panel. Commenting requires signing in with a `@ucsc.edu` Google account; the verified email is displayed as the comment author.
- **Sign-in button on both pages**, with a shared session (sign in once, signed in everywhere).

Videos live in a shared Google Drive folder; the catalog lives in a shared Google Sheet. Students add videos themselves by uploading to the folder and adding a sheet row. There is no admin panel, no database, and no moderation queue.

## Tech stack and constraints

- **Cloudflare Worker**, developed with `wrangler` (^4, Node 20+). Local development only: `wrangler dev` on **port 8787** (pin it in `wrangler.toml` under `[dev]`; the Google OAuth client ID is registered for `http://localhost:8787`, so the port must not drift).
- **No framework, no build step, no bundler, no database, no caching layer.** The sheet is the CMS, Drive is the CDN.
- **Frontend: plain HTML/CSS/JS** served as static assets by the same Worker via the `[assets]` binding in `wrangler.toml`. Exact asset paths are served first; anything else (`/api/*`, `/proxy/*`, `/watch`) falls through to the Worker's fetch handler. Same-origin throughout, so no CORS handling is needed.
- **No Google client libraries.** All Google auth is done by hand: an RS256 JWT signed with WebCrypto for the service account, and direct JWKS verification for ID tokens. The Workers runtime has WebCrypto, `fetch`, `atob`, and `TextEncoder`/`TextDecoder` — that is all you need.
- Dark "cinema" theme; responsive (grid collapses to 2 columns on tablets, 1 on phones).

## File layout

```
wrangler.toml        # Worker config: [assets] binding + [dev] port
.dev.vars            # secrets (gitignored); placeholders committed
src/index.js         # the entire Worker: endpoints + Google auth
public/index.html    # home page (gallery grid + header sign-in)
public/watch.html    # watch page (player + comments + sign-in)
public/gallery.js    # grid rendering + in-browser thumbnail capture
public/watch.js      # player + comment load/post
public/auth.js       # shared sign-in module (used by both pages)
public/style.css     # dark theme
README.md            # must include the full admin setup section (see below)
```

## Google credentials — two separate ones

**1. Service account (server-side).** Reads the Sheet, streams video from Drive, creates the Drive comments. Scopes: `https://www.googleapis.com/auth/spreadsheets.readonly` + `https://www.googleapis.com/auth/drive`. Auth: build a JWT `{iss: client_email, scope, aud: "https://oauth2.googleapis.com/token", exp, iat}`, sign it RS256 with WebCrypto (`crypto.subtle.importKey("pkcs8", ...)`, `RSASSA-PKCS1-v1_5`, SHA-256), POST it to `https://oauth2.googleapis.com/token` with `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer`, and cache the returned access token in module memory until ~60s before expiry.

**2. OAuth client ID (browser sign-in only).** Used with Google Identity Services (`https://accounts.google.com/gsi/client`). This is what students sign in with. It is *not* a secret — it is served publicly via `GET /api/config`.

Environment variables in `.dev.vars` (gitignored; the repo ships empty placeholders):

- `GOOGLE_CLIENT_EMAIL` — service account email
- `GOOGLE_PRIVATE_KEY` — PEM key; **keep the literal `\n` escapes**
- `GOOGLE_SHEET_ID` — from the sheet URL
- `GOOGLE_DRIVE_FOLDER_ID` — from the folder URL
- `GOOGLE_CLIENT_ID` — OAuth client ID (may be empty; sign-in UI shows a "not configured" note until it is set)
- Alternative to the first two: `GOOGLE_SERVICE_ACCOUNT_JSON` — the entire downloaded key file on one line.

## Sign-in flow (UCSC only)

1. Both pages call `vgAuth.init(...)` from `public/auth.js`.
2. `auth.js` fetches `/api/config` for the client ID, loads the Google Identity Services script, and calls `google.accounts.id.initialize({ client_id, hosted_domain: "ucsc.edu", callback })` then `renderButton` (dark theme, pill shape).
3. **`hosted_domain` is a UI hint only.** The real enforcement is server-side.
4. On credential receipt, `auth.js` POSTs the ID token to `/api/session`. The Worker verifies:
   - JWT decodes to 3 parts; `exp` in the future;
   - `iss` is `accounts.google.com` (or `https://accounts.google.com`);
   - `aud` equals `GOOGLE_CLIENT_ID`;
   - `email_verified` is true;
   - **signature** against Google's JWKS (`https://www.googleapis.com/oauth2/v3/certs`, cached ~1h in module memory, refetch once on unknown `kid`);
   - email ends with `@ucsc.edu`, and the `hd` claim (when present) equals `ucsc.edu`.
5. On success, `{ email, idToken }` is stored in `localStorage` (key `vg.auth`) and both pages show "Signed in as <email>" with a Sign out button. On each page load the saved token is re-verified server-side; if it fails (expired), it is cleared and the button reappears.
6. Posting a comment sends `idToken` in the body. If the server rejects it with 401 (expired token), the client signs out and forces re-sign-in.
7. Sign out clears `localStorage` and calls `google.accounts.id.disableAutoSelect()`.

## Video proxy (critical — do not skip or "simplify" this)

`GET /proxy/video/:id` streams the file from Drive: `https://www.googleapis.com/drive/v3/files/<id>?alt=media&supportsAllDrives=true`, using the service account access token, **forwarding the incoming `Range` request header** and passing back Drive's `Content-Range`, `Content-Length`, `Accept-Ranges`, and `Content-Type` headers. Return the same status Drive returned (206 for range requests).

Why this matters:

- **Seeking** works like a normal video site (Range requests).
- **Thumbnails** work at all: the home page streams the video through the proxy into a hidden `<video>` element, seeks to the sheet's timestamp, and draws a frame onto a `<canvas>` (data URL becomes the card thumbnail). Canvas is only untainted for same-origin resources — without the proxy, thumbnails would silently fail with a security error.
- Videos must be **MP4 (H.264 + AAC)** — browsers will not play the typical Drive/phone formats. The admin setup and student guide must say this.

## Sheet format (the CMS)

- Column A: Drive link (accept any common format: `/file/d/<ID>/view`, `?id=<ID>`, or a raw ID).
- Column B: thumbnail timestamp, `m:ss` (e.g. `1:23`).
- Column C: optional title; falls back to "Video" if blank.
- Gallery order = sheet row order. The page re-fetches on reload; tell users to refresh after editing the sheet.
- Parse `m:ss` or `h:mm:ss` into seconds; blank = frame 0.

## Endpoints (all same-origin)

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/config` | `{ clientId, ucscDomain }` — `clientId` is `null` when unset (never return `""`) |
| POST | `/api/session` | `{ credential }` → `{ email }` or 401 with a readable error |
| GET | `/api/videos` | Sheet rows parsed into `{ fileId, timestamp, title }` + `sheetId` for the "edit the sheet" link |
| GET | `/api/videos/:id/comments` | Drive comments on the file, parsed into `{ id, createdTime, author, content }` |
| POST | `/api/videos/:id/comments` | Body `{ content, idToken }`; verifies the token, stores `email: text` as a Drive comment, returns 201 with the created comment |
| GET | `/proxy/video/:id` | Range-capable video stream from Drive |
| GET | `/watch?id=<fileId>` | Watch page (missing `id` → redirect `/`) |

Comment storage format: `email: text` as the Drive comment content. When listing, split on the first `: ` (up to the first newline) to recover the author; keep the raw content as fallback text. The email is the displayed author name — there is no free-text name field on the form.

## wrangler.toml

```toml
name = "demo-video-gallery"
main = "src/index.js"
compatibility_date = "2026-05-01"

# The Google OAuth client ID is registered for http://localhost:8787,
# so the dev server must stay on this port.
[dev]
port = 8787

# The frontend (public/) is served as static assets by the same Worker:
# exact asset paths are served first; anything else (/api/*, /proxy/*, /watch)
# falls through to the fetch handler.
[assets]
directory = "./public"
binding = "ASSETS"
```

Note: `port` must be inside `[dev]` — a top-level `port` key produces a wrangler warning ("Unexpected fields found in top-level field: port").

## Deliverables the agent must produce

1. The complete project above, with **empty placeholders** in `.dev.vars` (never real keys).
2. A README that includes the **entire admin setup section** — reproduce the steps from [admin-setup.md](https://turkeubacon.github.io/demo-clip-info-site/admin-setup.md): Google Cloud project, enabling APIs, service account, sharing the folder and sheet, OAuth consent screen, OAuth client ID, where to find sheet/folder IDs, and where every value goes in `.dev.vars`. Do not summarize or skip steps; the admin may not know Google Cloud.
3. A short student guide (upload MP4 to the shared folder → add link + timestamp + title to the sheet → refresh; sign in with `@ucsc.edu` to comment).
4. A "How it works" section explaining the sheet-as-CMS, the proxy, thumbnail capture, and sign-in verification.
5. A troubleshooting section: "Malformed ID token" / "File not found" errors come from bogus IDs, not bugs; first sign-in failure usually means the origin isn't registered exactly (`http://localhost:8787`, no trailing slash) or the client ID has a typo; if the OAuth app is in Testing mode, sign-in only works for listed test users.

## Common pitfalls (learned the hard way)

- **Conflating the two credentials.** The service account key goes in `.dev.vars`; the OAuth client ID is also in `.dev.vars` but is *public* and only used in the browser.
- **Trusting the client-side JWT.** Decoding an ID token without verifying the signature is a security hole. Always verify against JWKS server-side.
- **Forgetting `supportsAllDrives=true`** on Drive API calls — shared-folder files fail without it.
- **Serving the video cross-origin** — breaks canvas thumbnails silently.
- **Returning `""` instead of `null` for an unset client ID** — the client would try to initialize Google sign-in with an empty client ID.
- **Not forwarding Range headers** — video seeking breaks.
- **`.dev.vars` is not hot-reloaded** by `wrangler dev` — restart the server after editing it (config in `wrangler.toml` *is* hot-reloaded).
- **Wrangler on non-standard Linux setups** — if `workerd` fails to launch, use whatever wrapper the local platform requires; the project itself is plain Node 20+ + wrangler and runs normally on mainstream systems.
