# Security notes

## 1. The passcode vault

Everything sensitive on a device lives inside a single encrypted blob in
`localStorage['hcs_secure_v1']`:

| Field | What it is |
|---|---|
| `firebaseToken` | the credential the app needs to read and write the database |
| `bank` | payment details (bank, sort code, account number) per teacher |

- The key is derived from the user's passcode with **PBKDF2-SHA256, 250,000
  iterations**, over a random 16-byte salt.
- The blob is encrypted with **AES-GCM** (random 12-byte IV per write).
- AES-GCM is authenticated, so a wrong passcode fails to decrypt rather than
  returning garbage — that is what the passcode check is.
- The derived key is held in memory only while unlocked. The passcode itself is
  never stored.

### Why this actually stops edits

The Firebase credential is *inside* the vault. While the app is locked
`getFbToken()` returns `''`, so `getFbUrl()` builds a URL with **no `auth`
parameter** — every request goes out unauthenticated and the database rules
reject it.

This is the important property: the lock is enforced by the server, not by
hiding buttons. Someone who opens the page without the passcode cannot read or
write, no matter what they do in the console.

### Lock behaviour

- **Auto-lock** after 30 minutes with no click, keypress or touch.
- **Lock now** button in the header.
- **Backoff** after 5 wrong attempts, doubling up to 60 seconds, which makes
  brute forcing impractical against a reasonable passcode.

### First run on each device

The lock screen picks its mode automatically:

| Situation | Mode | What you do |
|---|---|---|
| Vault already on this device | **Unlock** | Enter the passcode |
| Older build, plaintext token present | **Protect** | Choose a passcode — the existing token and payment details are encrypted and the plain copies deleted |
| Nothing here yet | **Set up** | Paste the database token, choose a passcode |

Each device has its own vault and its own passcode. There is no passcode
recovery: the passcode *is* the key. If it is lost, clear the browser's site
data for the page and set the device up again with the database token, which you
can always re-copy from the Firebase console.

## 2. Payment details never leave the device

Bank details are not part of the synced data.

- `syncToDb()` runs `stripBankFields()` over the teacher list before every push.
- `loadFromDb()` runs `absorbBankFields()`, which rescues any details still in
  the shared database into the vault and then strips them.
- All three write paths — the People modal, the Settings teacher form, and the
  import tool — go through the vault.
- The invoice generators read them at export time.

You therefore enter payment details once per device. Where they are not set,
invoices generate normally with the payment box omitted, and the preview says so.

## 3. Firebase database rules

The database URL is hardcoded in `index.html`, which is public. The URL is not a
secret — **the rules are the access control.**

Before this work the database was readable by anyone with no authentication,
exposing student names, attendance history and the bank details of two teachers.

`database.rules.json` closes it:

```json
{ "rules": { ".read": false, ".write": false } }
```

### Applying the rules

1. Firebase console → your project → **Realtime Database** → **Rules**
2. Paste the contents of `database.rules.json`
3. **Publish**

The app keeps working because it authenticates with the token from the vault,
and a legacy database secret bypasses rules.

### Before you publish — check every device

Every device needs the token in its vault. A device that has never had it will
show the **Set up** screen and ask for it. Have the token to hand before you
publish the rules.

### Verifying it worked

This should return `Permission denied` rather than data:

```bash
curl -s "https://hcs-dance-register-default-rtdb.europe-west1.firebasedatabase.app/hcs.json?shallow=true"
```

## 4. Crawlers

`robots.txt` disallows everything and `index.html` carries
`noindex,nofollow,noarchive,nosnippet`. This keeps well-behaved crawlers out of
the page. It is a courtesy measure, not a control — the passcode and the database
rules are what actually protect the data.

## 5. Rotate the exposed details

The database was publicly readable for some time, so treat what was in it as
potentially seen. The exposed bank details were a sort code and account number —
the same details printed on any invoice — so the harm is low, but the two
affected teachers should be told. Student names and attendance were also
readable; if the school has a data protection lead, that is worth a note.

## 6. Known limitation — the local cache is not encrypted

The vault covers the database credential and the payment details. It does **not**
cover `localStorage['hcs_v4']`, which holds the cached register: student names,
teacher names and attendance.

So the passcode:

- **does** stop edits — no credential means the database rejects every write,
  and that is enforced by Firebase, not by the UI;
- **does** stop casual viewing — the lock screen covers the whole app;
- **does not** stop someone who already has the device unlocked and knows how to
  open developer tools from reading the cached register.

Closing that gap means encrypting the whole cache behind the same passcode. It is
a bigger change — every save becomes an encrypt — and worth doing if the register
is ever kept on a shared or easily-lost device. Until then, device-level
protection (screen lock, disk encryption) is what covers it.
