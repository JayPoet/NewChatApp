# Locked Room — private link-based chat

A single-page, no-login chat app for a small group of friends. Anyone with the
link (and, if you turn it on, a 4-digit PIN) can pick a name and start
chatting in real time. Message text is encrypted client-side with a shared
passphrase before it ever reaches Firebase.

This is a genuinely small deployment — one Firebase project, one static HTML
file, no server to run. Everything below assumes zero prior Firebase
experience.

## What's in this folder

| File | Purpose |
|---|---|
| `index.html` | The entire app — HTML, CSS, and JS in one file. |
| `firestore.rules` | Database access rules. |
| `storage.rules` | File-upload access rules. |

## 1. Create a Firebase project

1. Go to [console.firebase.google.com](https://console.firebase.google.com) → **Add project**. Name it anything (e.g. `locked-room`). You can skip Google Analytics.
2. In the left sidebar: **Build → Authentication → Get started**. Click the **Anonymous** provider and enable it. This is what lets people join with no email/password.
3. **Build → Firestore Database → Create database**. Choose **production mode** and pick a region close to your group.
4. **Build → Storage → Get started**. Same region, production mode.
5. In **Project settings** (gear icon, top left) → scroll to "Your apps" → click the `</>` (web) icon → register an app (any nickname). Firebase will show you a config object that looks like:

```js
const firebaseConfig = {
  apiKey: "AIza...",
  authDomain: "locked-room-xxxx.firebaseapp.com",
  projectId: "locked-room-xxxx",
  storageBucket: "locked-room-xxxx.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abcdef"
};
```

6. Open `index.html`, find the `firebaseConfig` object near the top of the `<script>` block, and paste your real values in over the `REPLACE_ME` placeholders.

This `apiKey` is not a secret — it's fine to ship it inside the HTML. What actually protects your data is the security rules in step 2 below, plus the client-side encryption.

## 2. Apply the security rules

In the Firebase console:

- **Firestore Database → Rules** tab → replace the contents with everything in `firestore.rules` → **Publish**.
- **Storage → Rules** tab → replace the contents with everything in `storage.rules` → **Publish**.

These rules require every reader/writer to be signed in (anonymously — the app does this automatically) and restrict who can edit room settings or kick members to whoever the app marked as admin.

## 3. Deploy the frontend

Since it's one static file, any static host works. Netlify's drag-and-drop is the fastest:

1. Go to [app.netlify.com/drop](https://app.netlify.com/drop).
2. Drag `index.html` onto the page.
3. Netlify gives you a URL like `https://random-name-123.netlify.app`.

Or with Vercel:

```bash
npm i -g vercel
vercel --prod ./   # from the folder containing index.html
```

Or Firebase Hosting, if you'd rather keep everything in one place:

```bash
npm i -g firebase-tools
firebase login
firebase init hosting   # choose the project you made above, public dir = this folder
firebase deploy
```

## 4. Generate your room link and invite people

Open your deployed URL with no hash — the app will generate a random room ID
and update the address bar to something like:

```
https://your-app.netlify.app/#/chat/8f3a1c2e-...
```

**That full URL — hash included — is the invite link.** Send it to your
group along with:

- The **room passphrase** you want everyone to use (this is never sent to
  Firebase; it only lives in each person's browser and is used to derive the
  encryption key). Pick something the group can remember or share once
  over a call/Signal message — not over the same chat you're protecting.
- The **PIN**, if you turned that on in the admin panel (⚙️ → Room settings).

Whoever opens the link first automatically becomes the room admin.

## Environment variables

This app has no build step, so there's no `.env` file — configuration lives
directly in `firebaseConfig` inside `index.html`, as described in step 1.
If you later move this into a bundler (Vite, etc.), the natural mapping is:

```
VITE_FIREBASE_API_KEY
VITE_FIREBASE_AUTH_DOMAIN
VITE_FIREBASE_PROJECT_ID
VITE_FIREBASE_STORAGE_BUCKET
VITE_FIREBASE_MESSAGING_SENDER_ID
VITE_FIREBASE_APP_ID
```

## Data model

```
rooms/{roomId}
  name: string
  description: string
  adminId: string            // uid of the first person who ever joined
  pinEnabled: boolean
  pin: string | null
  retentionDays: number       // 0 = keep forever

rooms/{roomId}/users/{uid}
  username: string
  avatar: string               // an emoji
  status: string
  isAdmin: boolean
  online: boolean
  removed: boolean             // soft-kick flag
  lastSeen: timestamp
  joinedAt: timestamp

rooms/{roomId}/messages/{messageId}
  senderId, senderName, senderAvatar: string
  type: 'text' | 'image' | 'video' | 'audio' | 'document'
  text: string                 // AES-GCM ciphertext, base64 (type 'text' only)
  iv: string                   // AES-GCM IV, base64
  mediaUrl, fileName, fileSize
  replyTo: string | null       // messageId
  replyText: string | null     // plaintext preview, built client-side at send time
  reactions: { '👍': [uid, ...], ... }
  readBy: [uid, ...]
  isDeleted: boolean
  timestamp: server timestamp

rooms/{roomId}/typing/{uid}
  username: string
  ts: server timestamp         // clients treat entries older than 4s as stale
```

## How the encryption works

- The admin picks a passphrase and shares it with the group out of band.
- Each client derives an AES-256 key from that passphrase with PBKDF2
  (150,000 iterations, SHA-256) and a fixed, non-secret salt baked into the
  app.
- Message text is encrypted with AES-GCM before it's written to Firestore;
  a fresh random IV goes alongside each message.
- Firebase (and anyone who somehow got read access to your Firestore data
  without the passphrase) sees only ciphertext.
- **This protects message text from the backend and from network
  eavesdroppers, but it is not a substitute for real security auditing.**
  Media files (images/video/audio/documents) are currently uploaded
  unencrypted to Storage — treat this as a private-group convenience tool,
  not a tool for anything genuinely high-stakes.
- If someone loses the passphrase, older messages are unrecoverable by
  design — there's no backdoor.

## Known limitations / what's simplified vs. the original spec

To keep this a single deployable file with no build pipeline, a few things
are intentionally lighter-weight than a full production chat product:

- **Video** is uploaded as-is rather than transcoded/compressed
  client-side — the 100MB cap is enforced, but there's no FFmpeg step.
- **Push notifications** use the basic browser Notification API while the
  tab is open in the background; there's no service-worker-based push for
  when the tab/browser is fully closed.
- **Offline caching** isn't implemented — if you go offline, you'll see a
  connection drop rather than cached history (Firestore's own offline
  persistence could be enabled later if you want this).
- Rate limiting / abuse prevention beyond Firebase's defaults isn't
  included — fine for a handful of trusted friends, not for a public link
  you'd post anywhere.

## Testing checklist

- [ ] Open the link in two different browsers, join with two usernames, confirm messages sync instantly
- [ ] Send an image, a short audio note, and a small PDF — confirm they upload and display
- [ ] Close one tab, confirm the other shows that member going offline within ~30s
- [ ] Reply to a message, react to a message, confirm both show up for the other user
- [ ] Search for a word that appears in one message, confirm only matches show with highlighting
- [ ] Open ⚙️ → Room settings as the first-joined user, rename the room, kick a test member
- [ ] Toggle dark/light mode and confirm it persists on reload
- [ ] Export chat history and confirm the downloaded JSON has decrypted text
