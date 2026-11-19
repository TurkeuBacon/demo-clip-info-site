# Demo Gallery — Admin Setup Guide

Everything you need to configure before the gallery works. Budget about 15 minutes. All of this happens in Google Cloud plus one file in the project.

## 1. Create a Google Cloud project

1. Go to <https://console.cloud.google.com>.
2. Create a project (or pick an existing one). Note the project name — you'll use it for everything below.

## 2. Enable the APIs

**APIs & Services → Library**, then enable both:

- **Google Drive API**
- **Google Sheets API**

## 3. Create the Drive folder and the Sheet

1. In Google Drive, create a folder (e.g. "Demo Videos"). This is where students upload videos.
2. Create a Google Sheet (e.g. "Demo Gallery") with columns: **A = Drive link, B = thumbnail timestamp, C = optional title**.
3. Get the two IDs from the URLs:
   - Folder: `https://drive.google.com/drive/folders/<FOLDER_ID>`
   - Sheet: `https://docs.google.com/spreadsheets/d/<SHEET_ID>/edit`

## 4. Create a service account

1. **APIs & Services → Credentials → Create credentials → Service account.** Name it (e.g. `demo-gallery`), skip the optional role/keys steps.
2. Open the service account → **Keys → Add key → Create new key → JSON**. A JSON file downloads — this is the only time you can get it, keep it safe.
3. Note the `client_email` value inside it (looks like `demo-gallery@<project>.iam.gserviceaccount.com`).

## 5. Share the folder and sheet with the service account

The service account is a real Google identity — it needs access to both resources:

- **The Drive folder** → Share → paste the service account email → role **Commenter** (Commenter is enough for comments; Viewer is enough if you don't want file comments).
- **The Sheet** → Share → paste the service account email → role **Viewer**.

## 6. Create the OAuth client ID (for sign-in)

1. **APIs & Services → OAuth consent screen** → User type **External** → fill in app name and your email → save. No scopes need to be added manually; sign-in uses the non-sensitive `openid/email/profile` defaults.
2. Decide who can sign in:
   - **Testing mode** (default): only accounts listed as **Test users** can sign in. Add your UCSC email (and any testers) under **Audience → Test users**.
   - **Production mode**: click **Publish app** — then any `@ucsc.edu` account can sign in. Google won't require verification since only basic scopes are used; users just see a small "unverified app" notice.
3. **APIs & Services → Credentials → Create credentials → OAuth client ID → Web application.** Name it anything.
4. Under **Authorized JavaScript origins** add exactly:
   ```
   http://localhost:8787
   ```
   (No trailing slash. Add `https://your-domain` later if you deploy.)
5. **Authorized redirect URIs**: leave empty — not needed.
6. Copy the client ID (looks like `1234567890-abc123.apps.googleusercontent.com`).

## 7. Fill in `.dev.vars`

Open `.dev.vars` in the project and replace the placeholders:

```
GOOGLE_CLIENT_EMAIL=<service account email from step 4>
GOOGLE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"
GOOGLE_SHEET_ID=<SHEET_ID from step 3>
GOOGLE_DRIVE_FOLDER_ID=<FOLDER_ID from step 3>
GOOGLE_CLIENT_ID=<client ID from step 6>
```

Notes:

- **Keep the double quotes and the literal `\n` escapes** in the private key exactly as shown (copy everything between the quotes from the JSON file's `private_key` field).
- Alternatively, paste the entire downloaded key file on one line as `GOOGLE_SERVICE_ACCOUNT_JSON=...` instead of the email + key pair.
- `.dev.vars` is gitignored — it will never be committed.

## 8. Run it

```bash
cd demo-video-gallery
npm install   # Node.js 20+
npm run dev   # → http://localhost:8787
```

If you change `.dev.vars`, restart the dev server — it does not hot-reload that file.

## 9. Verify

1. Open <http://localhost:8787> — the gallery should show your sheet's rows.
2. Click a video — it should play (streamed through the proxy).
3. Click **Sign in with Google** and sign in with a `@ucsc.edu` account. Google's account picker will only offer UCSC accounts.
4. Post a comment — it appears with your UCSC email as the author, and also on the file in Drive.

**If something's off:** first sign-in failure usually means the origin isn't registered exactly (`http://localhost:8787`, no trailing slash) or the client ID in `.dev.vars` has a typo. If Google shows "Access blocked", your account isn't a test user (or the app needs publishing). "File not found" in the server logs means the file ID in the sheet is wrong or not shared with the service account.

## 10. The student workflow

Tell students:

1. Upload your video to the shared Drive folder as **MP4 (H.264 + AAC)** — what browsers play natively.
2. Add a row to the shared sheet: your Drive link, a thumbnail timestamp (`m:ss`), and optionally a title.
3. Refresh the gallery page.
4. Sign in with your UCSC account to comment on other people's videos.
