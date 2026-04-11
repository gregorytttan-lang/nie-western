# NIE Western — QR Ordering System

## Files
| File | Purpose |
|------|---------|
| `index.html` | Customer QR menu (GitHub Pages) |
| `kitchen.html` | Kitchen order dashboard (tablet) |

---

## Setup Steps

### 1. Create Firebase Project
1. Go to https://console.firebase.google.com
2. Create new project — name it e.g. `nie-western-pos`
3. Enable **Realtime Database** (Singapore region: `asia-southeast1`)
4. Set rules to allow read/write (test mode for now):
```json
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```
5. Go to Project Settings → Web App → Register app → copy config

### 2. Add Firebase Config
In **both** `index.html` and `kitchen.html`, replace the placeholder block:
```js
apiKey:            "YOUR_API_KEY",
authDomain:        "YOUR_PROJECT_ID.firebaseapp.com",
databaseURL:       "https://YOUR_PROJECT_ID-default-rtdb.asia-southeast1.firebasedatabase.app",
projectId:         "YOUR_PROJECT_ID",
storageBucket:     "YOUR_PROJECT_ID.appspot.com",
messagingSenderId: "YOUR_SENDER_ID",
appId:             "YOUR_APP_ID"
```

### 3. Deploy to GitHub Pages
1. Create new GitHub repo e.g. `nie-western`
2. Push both files
3. Settings → Pages → Deploy from main branch
4. Your URL: `https://yourusername.github.io/nie-western/`

### 4. Generate QR Code
- QR points to: `https://yourusername.github.io/nie-western/`
- Kitchen opens: `https://yourusername.github.io/nie-western/kitchen.html`

### 5. Install RawBT on the RK3566 tablet
- Install RawBT from Play Store
- Connect Xprinter via USB
- In RawBT: Settings → Printer → USB → select your Xprinter
- Test print from RawBT first

### 6. Open Kitchen Page
- Open `kitchen.html` on the tablet browser
- Keep screen on (Settings → Display → Screen timeout → Never)
- Kitchen page auto-refreshes via Firebase realtime

---

## How It Works
1. Customer scans QR → `index.html` opens on phone
2. Customer selects items → adds table number → places order
3. Order writes to Firebase instantly
4. Kitchen tablet (`kitchen.html`) receives order in real-time
5. Staff prints receipt via RawBT (USB to Xprinter)
6. Staff taps ✓ Done when order is ready

## Breakfast Hours
- Breakfast category **auto-hides after 11:15 AM**
- No manual action needed

## Notes
- Order numbers cycle 01–99 then reset
- Done orders shown faded at bottom; use "Clear Done" to remove
- Reopen button available if order accidentally marked done
