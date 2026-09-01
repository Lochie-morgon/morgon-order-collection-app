# Claim Tag — GitHub Pages + Firebase version

A standalone version of Claim Tag: the page is hosted on GitHub Pages, and
the shared pickup records live in Firebase (Firestore). No claude.ai
account needed to use it — anyone with the link can look up an order
number and confirm collection.

## What's in this folder
- `index.html` — the whole app (one file, no build step).
- `firestore.rules` — the security rules that actually enforce the lock
  (see below — this file matters as much as `index.html`).
- `README.md` — this file.

## Why this is safer than the plain-GitHub version

Firebase's web config (the block you'll paste into `index.html`) is *not*
a secret — Google's own docs say it's fine to ship in client-side code,
the same way a website's URL isn't a secret. Unlike the earlier
GitHub-file approach, there's no access token embedded in the page that
someone could steal and use to deface anything. The actual security
boundary is `firestore.rules`, enforced by Firebase's servers on every
read and write — not by hiding anything in the page.

That said, this app still has no login system — anyone with the link can
confirm a pickup, same as every version of Claim Tag so far. The rules
below stop someone from tampering with existing records or writing
garbage data, but they don't stop a stranger with the link from
confirming a real order. If that ever becomes a problem, the fix is
adding real authentication (e.g. Firebase Auth with an email allowlist),
which I'm happy to help with if it comes up.

## How the locking actually works

Each order number is its own document in a `records` collection (e.g.
`records/ABC123`). `firestore.rules` allows a document to be **created**
but never **updated or deleted**. Firestore treats the first write to a
given document path as a "create" and every write after that as an
"update" — so the very first confirmation for a number succeeds, and
every attempt after that is rejected by the rules, automatically, even
if two people confirm the same number at the exact same instant.
(Firestore serializes writes to a single document, so there's no race
window.) The app also wraps each confirm in a transaction so the person
who loses the race sees a clear "someone else just confirmed this" message
instead of a raw error.

## Setup

### 1. Create a Firebase project
1. Go to https://console.firebase.google.com and create a new project
   (the free "Spark" plan is enough for this).
2. In the project, go to **Build → Firestore Database → Create database**.
   Choose a location close to your users and start in **production mode**
   (the rules file below replaces the default-deny rules).
3. Go to **Project settings** (gear icon) → **General** → scroll to
   "Your apps" → click the **</>** (web) icon → register an app (any
   nickname) → you don't need Firebase Hosting, just the config.
4. Firebase shows you a `firebaseConfig` object. Copy it.

### 2. Deploy the security rules
In the Firestore Database section of the console → **Rules** tab, paste
the entire contents of `firestore.rules` from this folder, replacing
whatever's there, and click **Publish**. (Alternatively, if you're
comfortable with the Firebase CLI: `firebase deploy --only firestore:rules`
from a folder containing this `firestore.rules` and a `firebase.json`
pointing to it.)

### 3. Configure `index.html`
Near the top of the `<script type="module">` block, replace the
`FIREBASE_CONFIG` placeholder with the real config object from step 1:
```js
var FIREBASE_CONFIG = {
  apiKey: "AIza...",
  authDomain: "your-project.firebaseapp.com",
  projectId: "your-project",
  storageBucket: "your-project.firebasestorage.app",
  messagingSenderId: "...",
  appId: "..."
};
```
Commit this change.

### 4. Host it on GitHub Pages
1. Create a GitHub repository (public or private — GitHub Pages sites are
   public either way, unless you're on GitHub Enterprise Cloud with
   private Pages).
2. Add `index.html` to the repo (the `firestore.rules` file doesn't need
   to be there too, but it doesn't hurt to keep it alongside for
   reference/version history).
3. Repo → **Settings → Pages** → Source: "Deploy from a branch" → pick
   your branch → `/ (root)` → **Save**.
4. GitHub gives you a URL like
   `https://your-username.github.io/your-repo-name/` — that's the link to
   share.

### 5. Test before sharing widely
Open the link, confirm a test order number, then open it again (or have
someone else open it) and confirm the *same* number is shown as already
collected.

## Costs

Firestore's free tier covers 50,000 reads and 20,000 writes a day, which
is far more than a parcel-pickup desk will use. You won't need to add a
billing method unless usage grows dramatically beyond that.

## If you ever want to restrict who can confirm

Right now, like every version of Claim Tag built so far, anyone with the
link can confirm a pickup — there's no login. If Morgon wants to restrict
that to staff only, the clean way is Firebase Authentication (e.g. a
Google sign-in restricted to your company domain, or an email allowlist),
with `firestore.rules` updated to check `request.auth != null` (or a
specific claim) before allowing a create. That's a meaningful follow-up
project, not a quick tweak — let me know if you'd like it.
