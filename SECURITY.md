# Security Notes — NIE Western POS

Last reviewed: 2026-09-07

This system is a set of static HTML pages served from GitHub Pages, talking
directly to a Firebase Realtime Database from the browser. That shape has one
hard consequence, and every item below follows from it:

> **There is no server. Anything the page can do, a stranger with the same URL
> can do. The only thing that can actually stop them is the Firebase Database
> Rules.**

---

## 1. The Firebase database is the whole security boundary

`index.html` is the customer QR menu. It is public by necessity — customers scan
a QR code and load it on their own phones. It contains the database URL:

```
https://nie-western-pos-default-rtdb.asia-southeast1.firebasedatabase.app
```

That URL is therefore public forever, and cannot be hidden. Making the GitHub
repo private does **not** help: the database is reachable directly, without the
repo and without the pages.

So the database rules are the only control. **If the rules are open, everything
below is readable and deletable by anyone in the world.**

### Current status — root read is DENIED (verified 2026-09-07)

An unauthenticated read of the database root returns:

```json
{ "error" : "Permission denied" }
```

So the database is **not** in the fully-open `".read": true` state at the root.
That rules out the worst case: a stranger cannot dump the entire database in one
request.

**This does not mean the database is secure.** A root read can be denied while
individual paths are still world-readable, because Firebase rules cascade
downward from wherever they are granted, not upward. The root test tells you
nothing about:

* whether `/orders`, `/archives`, `/payouts` or `/float` are individually open
* whether **writes** are allowed anywhere — the root test only tested reading

### Still to verify

Open each of these in an incognito window. `Permission denied` is the answer you
want; anything else means that path is public.

```
.../orders.json?shallow=true
.../archives.json?shallow=true
.../payouts.json?shallow=true
.../float.json
```

`/orders` is the one to check first. The kitchen tablet has no way to sign in
today, so for it to receive orders at all, `/orders` almost certainly has
`".read": true`. If so, every order, price and total placed today is readable by
anyone — and very likely deletable too.

**The definitive check** is not a URL test at all. Firebase Console → Realtime
Database → **Rules** tab shows the exact rule set. Reading that answers every
question above at once, including writes, which cannot be tested safely from a
browser address bar.

## Emergency interim rules — safe to apply while the stall is open

If the incognito test above showed the database is open, apply this **now**. It
does not require any code change and does not touch the order flow.

```json
{
  "rules": {
    "orders":        { ".read": true, ".write": true },
    "queue":         { ".read": true, ".write": true },
    "queue_display": { ".read": true, ".write": true },
    "stock":         { ".read": true, ".write": true },
    "availability":  { ".read": true, ".write": true },
    "stall_status":  { ".read": true, ".write": true },
    "special":       { ".read": true, ".write": true },

    "archives":      { ".read": false, ".write": false },
    "payouts":       { ".read": false, ".write": false },
    "float":         { ".read": false, ".write": false },

    "$other":        { ".read": false, ".write": false }
  }
}
```

**Keeps working:** customers ordering, kitchen receiving, printing, queue
display, stock, availability, opening and closing the stall. Nothing in the
serving flow reads or writes the three locked paths.

**Stops working, on purpose, until Firebase Auth is in place:**

* the Reports tab in Admin (revenue figures, payouts, float)
* the end-of-day archive and Sheets save

So do the auth work the same evening, before you close. If you need the closing
routine before then, set the three locked paths back to `true` for the few
minutes it takes, then lock them again.

**What this does not fix:** `orders` still has to stay open, because the kitchen
tablet has no way to identify itself yet. Today's orders remain readable and
deletable by anyone. Only Step 1 below closes that.

---

## Fix, in priority order

### Step 1 — Add Firebase Authentication (this is the real fix)

Staff devices need a real identity so the rules can tell staff apart from the
public. There is no way around this step.

1. Firebase Console → **Authentication** → Get started
2. Enable **Email/Password**
3. Add one account per staff device (e.g. `kitchen@niewestern.local`)
4. Add a sign-in step to the staff pages, replacing the `STAFF_PWD` check
5. Remove `STAFF_PWD` from all source files

### Step 2 — Apply locked-down rules

Apply these **together with Step 1** — applying them first will break the
kitchen tablet, because it has no way to sign in yet.

```json
{
  "rules": {
    "orders": {
      ".read": "auth != null",
      "$orderId": {
        ".read": true,
        ".write": "!data.exists() || auth != null",
        ".validate": "newData.hasChildren(['id','items','total'])"
      }
    },
    "archives":      { ".read": "auth != null", ".write": "auth != null" },
    "payouts":       { ".read": "auth != null", ".write": "auth != null" },
    "float":         { ".read": "auth != null", ".write": "auth != null" },
    "stock":         { ".read": true, ".write": "auth != null" },
    "availability":  { ".read": true, ".write": "auth != null" },
    "stall_status":  { ".read": true, ".write": "auth != null" },
    "special":       { ".read": true, ".write": "auth != null" },
    "queue":         { ".read": true, ".write": true },
    "queue_display": { ".read": true, ".write": "auth != null" },
    "$other":        { ".read": false, ".write": false }
  }
}
```

The intent: the public can place an order and read the menu state and the call
display. The public cannot list all orders, cannot read the money, cannot change
anything, and cannot write to any path not named here.

Test with the **Rules Playground** (Console → Rules → Playground) before saving:
simulate an unauthenticated read of `/archives` and confirm it is denied.

### Step 3 — Redeploy the Apps Script endpoint

Deploy a **new version** of the Apps Script (this issues a new `/exec` URL),
update `SHEETS_URL`, and have the script reject requests that do not carry a
shared secret. The old URL keeps working until you archive the old deployment,
so archive it.

### Step 4 — Take the old copies off the public site

`nie_western_v2.html`, `nie_western_v88.html` and `nie_western_quickpos.html` are
all served publicly by GitHub Pages and all contain the admin screen. Keep only
the version actually in use, and move the rest out of the Pages branch.

---

## Rules of thumb for this codebase

* Never put a password, PIN, or shared secret in an HTML or JS file. Anything
  shipped to a browser is public.
* Never set `".read": true, ".write": true` on a path holding money, sales
  history, or order data — not even "temporarily for testing".
* If a Firebase write fails with `PERMISSION_DENIED`, the fix is a narrower rule
  for that specific path, not opening the path to everyone.
