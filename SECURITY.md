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

### Which file is live, and which paths it actually uses

The tablet runs **`nie_western_v88.html`**. Its Firebase paths are:

`orders`, `queue`, `queue_display`, `availability`, `stall_status`, `archives`,
`payouts`, `float` — all covered by the applied rules.

Two corrections to earlier notes in this file, both from checking v88 rather
than v2:

* **v88 has no stock feature.** `stock` appears nowhere in it — that is a v2
  feature. The `stock` rule that was added is harmless but unused, and nothing
  was broken by its absence. The earlier "stock counting is silently failing"
  note applies to `nie_western_v2.html`, which is not the live file.
* **v88 has no `special` path either** — also v2 only.

### Genuinely missing rule: `board_soldout`

`nie-display.html` reads and writes `board_soldout` (lines 370 and 461) and
there is no rule for it, so the sold-out board on the display is denied. If that
display is in use, add:

```json
"board_soldout": { ".read": true, ".write": true },
```

### Dead files still published: `index.html` and `kitchen.html`

Both use a different path prefix — `nie_western/orders` and
`nie_western/order_counter` — which has no rule and never had one, so neither
page can reach the database. They are leftovers from the original README setup,
superseded by v88, and they are still served publicly by GitHub Pages.

## Rules for the current two-file setup

```json
"orders": {
  ".read":  "auth != null",
  ".write": "auth != null",
  "$orderId": {
    ".read":  true,
    ".write": true
  }
}
```

`$orderId` write stays open to the public deliberately. Two client behaviours
require it and neither can be narrowed from the rules alone:

* v2 fills the queue number in with a second write after the order is saved
  (`nie_western_v2.html:1933-1942`).
* `saveOrderConfirmed` retries a full `set()` up to three times on a 7s timeout
  (`nie_western_v2.html:4583`). If the first write lands but the acknowledgement
  is lost, the retry targets an order that now exists. A create-only rule would
  reject it and show the customer a failure for an order that is in fact saved.

What that leaves closed, which is the part that matters:

* **Listing orders needs a sign-in** — the sales history cannot be read.
* **Writing at `/orders` itself needs a sign-in**, so `ordersRef.remove()` —
  wiping every order at once — is refused to the public. The `.write` at the
  `orders` level is what gives staff back their bulk operations; the public
  falls through it to `$orderId`.
* `archives`, `payouts` and `float` stay staff-only both ways.

Residual: someone who already knows a specific order id can alter or delete that
one order. They cannot enumerate ids, because listing is denied.

## Customers and staff run different files — the root of the 2026-09-09 incident

The kitchen tablet runs `nie_western_v88.html`. The customer QR opens
**`nie_western_v2.html`**. Everything below follows from that split.

The final rules of 2026-09-08 allowed the public to create an order but not to
alter one:

```json
"$orderId": { ".read": true, ".write": "!data.exists() || auth != null" }
```

That was verified against v88, where `nextQ()` resolves before the record is
built, so the queue number is present on the single create. **v2 does it the
other way round** (`nie_western_v2.html:1933-1942`): it saves the order first,
then issues the number and fills it in with a second write —

```js
ordersRef.child(o.id).update({queueNo}).catch(()=>{});
```

— which the rule refused. The `.catch(()=>{})` discards the error, so nothing
surfaced anywhere. The customer's phone showed the number from its own local
variable while the stored record had none, and the kitchen rendered `#000`.

Relaxed to `".write": true` on `$orderId` on 2026-09-09 to restore service.
Reading the order list still requires a sign-in, so the sales history stays
closed; what is open again is altering or deleting a single order whose id you
already know.

### Consequences of the split still outstanding

* **v2 still contains `STAFF_PWD: "123456789"`** (line 431) and its Admin view is
  reachable by anyone. The rules are what protect the data now: an intruder
  reaching that screen gets empty tables, because v2 never signs in to Firebase
  and cannot read `orders`, `archives`, `payouts` or `float`. What they *can*
  still do is toggle item availability and tamper with `queue_display` and
  `board_soldout`, all of which remain world-writable for the display tablet.
* **The security work in #1 and #2 landed only in v88** — the file customers do
  not use. v2 has no Firebase sign-in at all.
* **Two files sharing one database is the underlying fault.** Any rule tight
  enough to be worth having has to be checked against both, and this one was
  checked against one. Consolidating on a single file is the durable fix.

## Status

Closed as of 2026-09-08, verified against the live database:

| Item | State |
|------|-------|
| Public read of `/orders` | **Closed** — verified `Permission denied` while signed out |
| Public write/delete of `/orders` | **Partly closed** — listing and wiping all orders need a sign-in; a single order stays writable by id (see the two-file section below for why) |
| Public read/write of `archives`, `payouts`, `float` | **Closed** — staff only, both directions |
| Public write of `stall_status` | **Closed** — public may read it, only staff may set it |
| `STAFF_PWD` in `nie_western_v88.html` | **Removed** — sign-in goes to Firebase Authentication |
| Staff session lost on every reload | **Fixed** — the session persists |

Still open, deliberately or not yet done:

| Item | Note |
|------|------|
| `queue`, `queue_display`, `availability`, `board_soldout` world-writable | Accepted. `nie-display.html` runs unattended with no sign-in and writes to all four. Nuisance risk only — no money data or order history is reachable. |
| Apps Script `SHEETS_URL` public | Anyone who reads the page source can POST rows into the sales spreadsheet. Needs a redeployment — see Step 3. |
| Sign-out button not deployed | Written and on the branch, not yet merged. Without it a tablet stays signed in indefinitely. |
| `nie_western_v2.html`, `nie_western_quickpos.html` still published | Both still carry the old `STAFF_PWD` in source. The password no longer grants anything — the rules require a Firebase account the old files never obtain — so an intruder reaches an empty screen. Dead weight worth deleting. |
| `index.html`, `kitchen.html` still published | Address `nie_western/orders`, a prefix no rule covers. Non-functional. |

## Do this now, while the stall is trading## Do this now, while the stall is trading

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

#### The nightly Google Sheets save is not a backup

`doSaveDailyToSheets()` (`nie_western_v2.html:3654`) posts a single summary row
per day: date, total revenue, PayNow and cash splits, order count,
dine-in/takeaway counts, float, payout total, cash in box, and the top five
items. Payouts go as their own rows.

Individual orders are never sent — only aggregates computed from them. The menu
and stock are not sent either. The Sheet can tell you what a day took; it cannot
reproduce an order.

`archiveMonth()` (`nie_western_v2.html:3913`) is likewise a summary, and it
writes to `archives/<monthKey>` **inside the same Firebase database** that is
currently world-writable. It is not off-site and does not survive a wipe.

So after a wipe: the daily totals in Google Sheets survive, which covers the
accounting figures. Every individual order, the monthly archives, payouts, float
and stock are gone. The JSON export in section A is the only thing that captures
the actual data.

### Before changing any rules: how to undo it

A rules change controls **who may access** data. It never reads, writes or
deletes the data itself, so no rules edit can lose your sales history.

To make it reversible in 10 seconds:

1. Select everything in the Rules box and copy it into a note.
2. Make the change and click **Publish**.
3. If anything misbehaves, paste the old text back and Publish again.

Firebase also keeps a version history of rules, so that is a second way back.
The worst realistic outcome is a screen in Admin failing to load until you
revert.

If you would rather change one line at a time, start with just:

```json
"archives": { ".read": false, ".write": true },
```

Publish, take a test order, confirm the kitchen receives it, then do `payouts`
and `float` the same way, and add `stock` last.

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

### Step 1 — Deploy the login (code is written, on this branch)

`nie_western_v88.html` now signs in against Firebase Authentication instead of
comparing to a password held in the file. `STAFF_PWD` is gone from the source.

What changed:

* The password box looks and behaves the same. Behind it,
  `signInWithEmailAndPassword` checks the password on Google's servers.
* The account name (`CONFIG.STAFF_EMAIL`) stays in the file. An account name is
  not a secret; the password never reaches the file.
* Firebase remembers the session, so the tablet survives a refresh. The old
  `_authed` flag lived in memory and was lost on every reload.
* The all-orders feed (`attachOrdersListener`) now starts on sign-in rather than
  on page load. Customers never needed it — their own order is tracked by id,
  and queue numbers come from the `queue` counter — and once the rules require a
  sign-in to list orders, an unsigned-in phone would be refused it anyway.

**Order of deployment matters.** Deploy the code FIRST, then the rules. The new
code works fine under the current rules (signing in is simply extra), so there
is no window where the stall is broken. Doing it the other way round breaks the
tablet immediately.

1. Firebase Console → **Authentication** → Get started → enable
   **Email/Password**.
2. **Users** tab → Add user. Email `staff@niewestern.local`, and a strong
   password — this is what staff will type from now on. Not `123456789`.
3. Replace `nie_western_v88.html` on the site with the version from this branch.
4. On the tablet, hard-refresh and sign in with the new password. Check the
   Kitchen screen fills, and place a test order.
5. Only once that works, apply the rules in Step 2.

### Step 2 — Apply the final rules

```json
{
  "rules": {
    "orders": {
      ".read": "auth != null",
      "$orderId": {
        ".read": true,
        ".write": "!data.exists() || auth != null"
      }
    },

    "archives":      { ".read": "auth != null", ".write": "auth != null" },
    "payouts":       { ".read": "auth != null", ".write": "auth != null" },
    "float":         { ".read": "auth != null", ".write": "auth != null" },
    "stall_status":  { ".read": true,           ".write": "auth != null" },

    "queue":         { ".read": true, ".write": true },
    "queue_display": { ".read": true, ".write": true },
    "availability":  { ".read": true, ".write": true },
    "board_soldout": { ".read": true, ".write": true }
  }
}
```

Why each block is shaped that way:

* **`orders`** — a customer may create their order (`!data.exists()`) and read
  that one order back to track it, but cannot list every order in the stall or
  alter one that exists. Cancelling and status changes live in `renderKitchen`,
  which is staff-only, so nothing customer-facing needs write access to an
  existing order.
* **`archives`, `payouts`, `float`** — staff only, both directions. This is what
  finally stops a stranger deleting the sales history.
* **`stall_status`** — the public must read it to see the closed screen; only
  staff may set it.

#### What stays world-writable, and why

`queue`, `queue_display`, `availability` and `board_soldout` are still open to
anyone. `nie-display.html` runs unattended on the display tablet with no
sign-in, and it *writes* to all of these — `board_soldout` and `availability`
when someone taps an item sold out (line 370-371), and `queue_display` when a
call times out (line 440-441). Locking them would freeze the display.

The residual risk is nuisance, not loss: someone could flag items sold out or
clear the call display. Staff see it immediately and can undo it in seconds. No
money data and no order history is reachable this way.

To close it properly later, give the display tablet its own staff sign-in — the
session persists, so it would be signed in once at setup. That is worth doing,
but it adds a way for an unattended screen to fail silently mid-service, so it
should be a deliberate change on a quiet day rather than part of tonight.

#### Known edge case

If a customer's order write succeeds on the server but the acknowledgement is
lost, a retry becomes a write to an order that now exists, and the rule refuses
it. The customer sees an error even though the kitchen has the order. This is
rare and loses nothing; it is the accepted cost of stopping strangers rewriting
existing orders.

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
