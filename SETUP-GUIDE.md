# Kanri — Setup & Deployment Guide

A free, serverless kanban board. Boards live in your team's own Google Drive. Installable as an app (PWA).

## Deploying v5 (production)

Upload **all six files** to your GitHub repo root: `index.html`, `sw.js`, `manifest.webmanifest`, `icon.svg`, `icon-192.png`, `icon-512.png`. GitHub Pages serves them as-is. After deploying, users can install Kanri like a native app: Chrome/Edge → browser menu → *Install Kanri*; iPhone → Share → *Add to Home Screen*.

### Fixed in v5
- **Archived cards were unviewable / Share sometimes closed instantly.** Both were one bug: any dialog opened *from another dialog* (board menu → Archived, menu → Share) was immediately closed by its parent. Fixed with a dialog sequence guard — archived cards can now be browsed, restored, or permanently deleted.
- **Templates vanished.** Marking a board as a template moved it to the Templates section, but clicking it only offered "create a copy." Template tiles now open a chooser: *Create a board from this template* or *Open & edit the template itself*.
- **Member pictures rendered square** in dialogs — avatars are now proper circles everywhere.
- **Invite errors are explained**: if Google reports missing permissions, the app tells you to sign out/in and re-approve consent; if you lack sharing rights it says to ask the owner.

### New in v5
- **Sessions renew invisibly.** The app refreshes your Google token during your own clicks, before it expires — during active use you should never see a sign-in prompt. (Google tokens last 1 hour by design; after long idle, one click on the "Sign in" pill reconnects. Zero-prompt-forever needs a server holding refresh tokens — impossible in a pure static app.)
- **Boards open instantly.** Every board keeps a local snapshot; opening paints immediately from it while the fresh copy syncs behind, and merges if you'd already started editing.
- **Move arrows on cards** — ◀ ▶ buttons (visible on hover, always on touch) move a card to the neighboring list without dragging.
- **Description Save/Cancel buttons** — editing a card's description now requires an explicit Save, with Cancel to revert.
- **Priorities** — Urgent/High/Medium/Low flags on cards, a priority sort per list, and a Priority column in exports.
- **My cards filter** — one tap shows only cards assigned to you.
- **Browser notifications** — enable via the 🔔 bell: an alert when a card becomes overdue, and a reminder **1 hour before** the due time for cards assigned to you. Honest limit: notifications fire while a Kanri tab is open; push to a closed browser requires a push server.
- **Help (?) button** with a concise usage guide, on both the home and board screens.
- **Share dialog polish** — the board link is highlighted in a colored box with a prominent Copy button and auto-selects on click; invitation emails include the app link.
- **PWA** — installable, with offline app-shell caching (your last-viewed boards remain viewable offline; edits require a connection to Google Drive).

### About simultaneous editing
Kanri deliberately does **not** lock cards while someone edits — locking requires a server to be reliable. Instead, every save auto-merges: per card the newer edit wins, and comments/activity from both sides are always kept, so parallel work can't destroy anything. Sync runs every ~8 seconds and instantly on tab focus.

## 0. Production update (v4) — what changed and what you must do

Your Client ID (`504273238808-…apps.googleusercontent.com`) is now built into the file, so your team never sees a setup screen. A Client ID is a public identifier that's safe to publish in a webpage — but treat the client *secret* (`GOCSPX-…`) as sensitive: never paste it anywhere, and the app never needs it.

**One required action:** this version uses the full Google Drive scope instead of the limited `drive.file` scope. That change is what makes shared boards and email invitations work (the old scope could only see files the app itself created, which is why teammates' shared links failed). Because the permission changed, everyone must sign in once more and approve the new consent screen. If Google blocks sign-in, add the Drive API scope on your OAuth consent screen and make sure teammates are listed as Test users (or publish the app).

**What's new:**
- **No more sign-in screen on refresh.** Your session is cached in the browser; reloading the page paints your boards instantly from a local index, with zero contact with Google. If the hourly token has expired, the app never interrupts you — it shows a small "Sign in" pill and a one-click reconnect; your unsaved edits are held safely until you do.
- **Invite teammates by email, from inside the app.** Share → enter an email, pick Editor or Viewer, and Google sends them a real notification email. The board appears for them when they open your app link. Members can be removed from the same dialog.
- **Board members & @tagging.** Everyone who joins a board shows up in a member strip in the board header. Assign cards from the member list, and tag people in comments with the @ Tag button — mentions render highlighted.
- **Bullet-proof loading.** Every board is normalized on load: missing fields are repaired, duplicate cards de-duplicated, and unsafe links stripped, so old or hand-edited files can't crash the UI.
- **Professional icon set** — hand-drawn SVG icons throughout instead of text glyphs and emoji.
- Sign out now returns to the start page; syncing runs every 8 seconds with automatic merging.

## 1. Deploy to GitHub Pages (2 minutes)

1. Create a new GitHub repository (e.g. `kanri`).
2. Upload `index.html` to it.
3. Repo → **Settings → Pages** → Source: *Deploy from a branch* → Branch: `main`, folder `/ (root)` → Save.
4. Your app is live at `https://YOURNAME.github.io/kanri/`.

The app already works at this point in **local mode** (boards saved in each person's browser). For team use with Google Drive, do the one-time setup below.

## 2. One-time Google setup (~10 minutes, free, no billing)

Because there is no server, the page talks to Google directly from the browser. That requires a free OAuth Client ID:

1. Go to **console.cloud.google.com** → create a project (any name, e.g. "Kanri").
2. **APIs & Services → Library** → enable **Google Drive API** and **Google Sheets API**.
3. **APIs & Services → OAuth consent screen** → External → fill in the app name and your email.
   - Under **Test users**, add the Gmail addresses of everyone on your team (up to 100).
   - (Optional, later) Click **Publish app** to remove the test-user list. Google may show an "unverified app" warning screen that users click through — normal for private tools.
4. **Credentials → Create credentials → OAuth client ID → Web application**:
   - **Authorized JavaScript origins**: add `https://YOURNAME.github.io` (no path, no trailing slash). Add `http://localhost:8000` too if you want to test locally.
5. Copy the Client ID (ends in `.apps.googleusercontent.com`).

**Give it to the app** in either of two ways:
- Paste it into the setup screen the first time you click "Google Drive" mode (each user does this once), **or better:**
- Open `index.html`, find the line `const BUILT_IN_CLIENT_ID = "";` and paste your ID between the quotes, then re-upload. Now your whole team skips the setup screen entirely. (A Client ID is a public identifier, not a secret — it's safe to embed.)

## 3. How team sharing works

- Each board is a file inside a **"Kanri Boards"** folder in the owner's Google Drive.
- To share a board: board menu **⋯ → Share with your team**. Share the file in Drive (Editor = can edit, Viewer = read-only), then send teammates the app link shown — it opens the board directly.
- Everyone edits the same Drive file. The app refreshes shared boards about every 20 seconds and warns before overwriting a teammate's newer save.

## 4. Exports

- **Export → Google Sheets**: creates a spreadsheet with three tabs — Cards, Comments, and a full Activity log — all with created/updated timestamps.
- **Export → CSV**: three CSV files, works even in local mode without sign-in.
- **Board menu → Download backup (.json)**: full raw backup you can re-import by placing it in the Drive folder.

## 5. Troubleshooting sign-in

**"Error 401: invalid_client"** — Google doesn't recognize the Client ID. Check, in order:
1. The pasted ID is the complete string ending in `.apps.googleusercontent.com` with no spaces or line breaks (the app now strips whitespace and validates this before sending).
2. You pasted the Client ID, **not** the client secret (secrets start with `GOCSPX-` — the app now detects and blocks this).
3. The credential type is **OAuth client ID → Web application** (not Desktop app, not an API key).
4. Freshly created clients can take a few minutes to a few hours to activate — wait and retry.
5. The client/project wasn't deleted. Re-copy it from **Console → APIs & Services → Credentials**.

To re-enter a wrong ID: the error toast now shows a **Fix Client ID** button that reopens the setup screen.

**Origin/redirect errors** — add your exact page origin (e.g. `https://yourname.github.io`, no path) under **Authorized JavaScript origins** on the OAuth client.

**"App isn't verified" screen** — add each teammate's email under **Test users** on the consent screen, or click Advanced → Continue, or publish the app.

## 6. Collaboration update (conflict fix + auto-merge)

**Fixed: false "conflict" warnings.** Earlier versions made two API calls per save (content, then filename metadata); the second call bumped Drive's file version, so the app mistook its own save for a teammate's edit and warned on every change. Saves are now a single atomic request — one save, one version bump, no self-conflicts.

**New: automatic merging.** When a teammate really does save while you're editing, the app no longer interrupts you. It merges the two versions card by card:
- Edits to *different* cards are all kept.
- If the *same* card was edited on both sides, the more recent edit wins its fields — but comments and activity history from both sides are always combined, so nothing anyone wrote is lost.
- Cards and lists added, moved, archived, or deleted on either side are respected (deletions are tracked internally so a deleted card can't be resurrected by a stale copy).

Boards also refresh every ~12 seconds while open, a small toast tells you *who* made the update, and if someone has view-only Drive access the app detects it and switches to a read-only mode instead of failing on save.

## 7. Features added in v3 (recap)

- **Card archiving** — archive instead of delete, with an instant Undo toast; browse, restore, or permanently delete from Board menu → Archived cards.
- **Link attachments** — attach URLs to cards (auto-adds https://, validates); shown on the card as 🔗 and included in exports.
- **Duplicate list** — copies a whole column including its cards.
- **Read-only awareness** — viewers can browse shared boards without save-error spam.
- **Quiet save retry** — a failed save (bad network) retries automatically in the background.

## 8. What changed in v4 (production build)

**Your Client ID is now built in** — teammates never see a setup screen; they just open the link and sign in.

**Instant, silent refresh.** Sessions are cached in the browser, so reloading the page goes straight to your boards with no Google sign-in check. Tokens renew quietly in the background about once an hour. Signing out returns to the start page.

**Sharing actually works now — important context.** The previous versions used Google's `drive.file` permission, which only lets the app open files each user's own browser created. That's why teammates got errors on shared links. v4 uses the full Drive scope, which fixes it, with two consequences you should expect:
- **Everyone (including you) will be asked to grant access again** the first time they sign in after this update. That's normal — the permission changed.
- The consent screen wording is broader ("see and manage your Drive files"). The app still only ever touches its own "Kanri Boards" folder and boards opened by link.

**Email invites.** Share → type teammate emails → choose Can edit / View only → Send. Google delivers an invitation email containing the board link; the recipient signs in with that address and the board opens. No manual Drive sharing needed.

**Members & tagging.** Everyone who opens a board appears in its member strip (avatars next to the board title — click it to invite more). Assign cards from the member list, and use the **@ Tag** button in comments to mention members (highlighted in green).

**Tighter sync.** Boards refresh every ~8 seconds, resync instantly when you switch back to the tab or your network reconnects, and an Offline indicator appears when you lose connection (edits save automatically once you're back). All concurrent edits go through the auto-merge engine — no conflict prompts.

**Professional polish.** Proper icon set throughout, no start-screen flash on load, member avatars, dark/light icon toggle.

### Production checklist
1. Replace `index.html` in your repo with this build.
2. In Google Cloud Console → OAuth consent screen: add all teammates as **Test users**, or click **Publish app** so anyone with the link can sign in (they'll click through an "unverified app" notice — normal for internal tools).
3. Confirm your GitHub Pages origin (e.g. `https://yourname.github.io`) is listed under Authorized JavaScript origins.
4. Everyone re-grants access on first sign-in (one time, because of the scope fix).

## 9. Features added in v2 (recap)

- **Dark mode** — ◐ button in the top bar; follows your system preference by default and remembers your choice.
- **Trello import** — Import button on the home screen. In Trello: board Menu → Print, export & share → Export as JSON, then choose the file. Brings over lists, cards, descriptions, labels (with colors), due dates, checklists, and comments; archived lists/cards are skipped. Kanri `.json` backups can be imported the same way.
- **Card cover colors** — color bar across the top of a card, set from the card's side panel.
- **Assignees** — assign a card to a teammate by name; initials show on the card and the name appears in Sheets/CSV exports.
- **Hide done filter** — one tap to hide completed cards.
- **Keyboard**: press `/` on a board to jump to search; `Esc` closes dialogs.
- **Smarter sign-in errors** — the app now tells you exactly which Google Cloud setting to fix instead of a generic failure.

## 10. Honest limitations to know about

- This is **not** a byte-for-byte clone of kan.bn — that project is a Next.js + PostgreSQL *server* application and cannot run on GitHub Pages. Kanri matches its feature set (boards, lists, cards, labels & filters, comments, activity log, templates, sharing) within a fully serverless design.
- Collaboration is **near-real-time** (20-second refresh + conflict warning), not live-cursor real-time like Trello. True real-time requires a server.
- Google sign-in tokens last ~1 hour; the app silently renews them, but after long idle periods a user may need to click "Sign in" again.
- "No maintenance" is mostly true: GitHub Pages and the Drive/Sheets APIs are stable, free services. The only realistic future task is if Google ever changes its sign-in library (historically rare, announced years in advance).
- Limits are Google's, not the app's: Drive free tier is 15 GB (a board file is a few KB, so effectively millions of cards), and API quotas are far beyond what a team's clicking can reach.
