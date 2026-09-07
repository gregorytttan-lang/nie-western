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

### Confirmed rules — every path is world read/write (verified 2026-09-07)

The live rule set is:

```json
{
  "rules": {
    "orders":        { ".read": true, ".write": true },
    "queue":         { ".read": true, ".write": true },
    "queue_display": { ".read": true, ".write": true },
    "availability":  { ".read": true, ".write": true },
    "stall_status":  { ".read": true, ".write": true },
    "archives":      { ".read": true, ".write": true },
    "payouts":       { ".read": true, ".write": true },
    "float":         { ".read": true, ".write": true },
    "special":       { ".read": true, ".write": true }
  }
}
```

There is no rule at the root, which is why a read of `/.json` returns
`Permission denied`. That denial is misleading: it only prevents fetching
everything in a single request. Every path that actually holds data is granted
`true` individually, so anyone on the internet can read and delete all of it by
naming the path:

* `/archives.json` — full monthly revenue history
* `/payouts.json` — supplier payout records
* `/float.json` — cash float
* `/orders.json` — every order, item, price and total

Writes are open on all of them too, so the same stranger can wipe the sales
history, inject orders into the kitchen, or set `stall_status` to closed during
service.

### Bug found while reviewing the rules: `stock` is missing

The code reads and writes `stock` (`nie_western_v2.html:2811-2825`), but there is
no `"stock"` entry in the rules. Unlisted paths default to denied, so **stock
counting is silently failing right now** — every write returns
`PERMISSION_DENIED`. Adding `stock` to the rules fixes it.

## Do this now, while the stall is trading

### A. Export a backup first (5 minutes, no risk)

Writes are open on `archives`, so the sales history can be deleted by anyone.
Take a copy you control before changing anything.

Use a laptop, not the tablet — the export menu is awkward on a small screen.

1. Go to <https://console.firebase.google.com> and sign in with the Google
   account that owns the project.
2. Click the **nie-western-pos** project.
3. In the left sidebar, click **Realtime Database**.
4. Click the **Data** tab (it sits next to Rules).
5. **Check you are at the root.** The top line of the data panel should read
   `nie-western-pos-default-rtdb.asia-southeast1.firebasedatabase.app` with
   `orders`, `archives`, `payouts` listed underneath. If you have clicked into a
   child node, click that top line to go back up. Export only saves the node you
   are currently looking at — this is the one step people get wrong.
6. Click the **⋮** (three dots) at the top-right of the data panel.
7. Choose **Export JSON**. The browser downloads a `.json` file.
8. Rename it with the date: `nie-western-backup-2026-09-07.json`.
9. Copy it somewhere that is not the Downloads folder — Google Drive, or email
   it to yourself. A backup on one laptop is not a backup.

**Verify it worked before moving on.** Open the file. It should be large and
contain `archives`, `orders` and `payouts`. If it says `null` or is only a few
lines, you exported a child node — go back to step 5.

Note: the **Backups** tab in the console is a paid (Blaze) feature. On the Spark
plan this manual export is the way to do it, so repeat it at month end.

#### If the console export will not work (phone-only fallback)

Reads are currently open, so you can save each path straight from the browser.
**Do this before applying the rules in section B** — that change switches these
reads off.

Open each URL and save the page (or copy the text into a note):

```
https://nie-western-pos-default-rtdb.asia-southeast1.firebasedatabase.app/archives.json
https://nie-western-pos-default-rtdb.asia-southeast1.firebasedatabase.app/orders.json
https://nie-western-pos-default-rtdb.asia-southeast1.firebasedatabase.app/payouts.json
https://nie-western-pos-default-rtdb.asia-southeast1.firebasedatabase.app/float.json
```

### B. Interim rules — closes the read exposure, nothing stops working

```json
{
  "rules": {
    "orders":        { ".read": true,  ".write": true },
    "queue":         { ".read": true,  ".write": true },
    "queue_display": { ".read": true,  ".write": true },
    "availability":  { ".read": true,  ".write": true },
    "stall_status":  { ".read": true,  ".write": true },
    "special":       { ".read": true,  ".write": true },
    "stock":         { ".read": true,  ".write": true },

    "archives":      { ".read": false, ".write": true },
    "payouts":       { ".read": false, ".write": true },
    "float":         { ".read": false, ".write": true }
  }
}
```

Two changes from what is live today:

* **`stock` added** — fixes the silently broken stock counting described above.
* **Read turned off on `archives`, `payouts`, `float`** — a stranger can no
  longer read your revenue history, payouts or cash float.

Writes stay on deliberately, so every staff action still succeeds mid-service:
recording a payout, setting the float, and the end-of-day archive save all
write, they do not read. The app writes to `localStorage` first and syncs to
Firebase second (`saveFloat`, `addPayout`), so staff devices keep showing their
own data from cache.

The only visible loss: the monthly archive list in the Reports tab will be
empty, and payouts recorded on the tablet will not appear on the phone until
tonight.

**Stricter option** — if no payouts or float changes happen during service, set
`".write": false` on those same three paths as well. That also blocks a stranger
from deleting the sales history. Recording a payout would then fail with
"Saved on this device only", which is handled but not ideal mid-shift.

**What this does not fix:** `orders` must stay open, because the kitchen tablet
has no way to identify itself yet. Today's orders stay readable and deletable by
anyone until Step 1 below is done.

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
