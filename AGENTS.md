# SPENT — Bill & Expense Tracker (Android PWA + Native APK)

## What This Is
Personal expense tracking app with bill management. PWA frontend + Android APK wrapper with FCM notifications. Tracks spending, manages recurring bills, supports partner (Ibu) approval flow.

## Key URLs
- **Staging:** https://azelinc.github.io/SPENT-staging/
- **Production:** https://azelinc.github.io/SPENT/
- **GitHub:** `azelinc/SPENT.git` (source: branches `main` + `SPENT-staging`)
- **Staging deploy repo:** `azelinc/SPENT-staging.git` (separate repo, `main` branch)
- **Project root (dev):** `/opt/data/home/spent-staging/` (source)
- **Project root (deploy):** `/opt/data/home/spent-staging-deploy/` (GitHub Pages deploy)
- **Project root (prod):** `/opt/data/home/spent-prod/`
- **Firebase project:** `ainvested-703ec`

## File Structure
- `sp7.js` — main PWA logic (SPENT-specific, 2100+ lines)
- `sp.js` — shared/invested utilities
- `sp7.css` — SPENT-specific styles
- `sp.css` — shared styles
- `index.html` — PWA shell
- `sw.js` — service worker
- `manifest.json` — PWA manifest
- `icon-192.png`, `icon-512.png` — app icons

## Deploy Flow
**Two-repo system (separate repos, not branches):**

| Env | Repo | Branch | URL |
|-----|------|--------|-----|
| Source | `azelinc/SPENT.git` | `SPENT-staging` | — |
| Staging | `azelinc/SPENT-staging.git` | `main` | `https://azelinc.github.io/SPENT-staging/` |
| Production | `azelinc/SPENT.git` | `main` | `https://azelinc.github.io/SPENT/` |

**Staging deploy flow:**
1. Edit files in `/opt/data/home/spent-staging/` (on `SPENT-staging` branch)
2. Run `python3 /opt/data/scripts/deploy_spent_staging.py` — commits source, copies files to `spent-staging-deploy/`, pushes to `azelinc/SPENT-staging.git main`
3. Or manually: commit to `SPENT-staging` branch, then sync deploy repo

**Production deploy flow:**
1. Changes merged from `SPENT-staging` → `main` in `azelinc/SPENT.git`
2. Push `main` → `pages-build-deployment` GitHub Actions auto-deploys

## Version Bump
- `sp7.js` line ~20: `APP_VER = 'vX.Y.Z'`
- `index.html`: update cache buster `?v=N` on script/link tags
- `sw.js`: update `CACHE_NAME` constant

## Database
- **Firebase RTDB** (same project as Invested, Expensed, Maintained)
- URL: `https://ainvested-703ec-default-rtdb.asia-southeast1.firebasedatabase.app`
- Expenses: `users/{uid}/expenses/{pushId}`
- Bills: `bills/{uid}/{pushId}`
- FCM tokens: `fcmTokens/{uid}/{tokenKey}/token`

## Expense Schema (RTDB)
```
{
  amount: number,
  category: string (e.g. "Gold", "Food", "Groceries", "Fuel"),
  subCategory: string (optional),
  merchant: string (optional),
  date: "YYYY-MM-DD",
  status: "approved" | "pending",
  notes: string (optional),
  timestamp: epoch_ms,
  paymentMethod: "Cash" | "Card" | "QR" (optional),
  type: "expense" | "income" (optional — absent = expense)
}
```

## Bills Schema (RTDB)
Path: `bills/{uid}/{pushId}`
```
{
  name: string,
  amount: number (optional, 0 = date-only tracking),
  dueDay: number (1-31),
  reminderDays: [7, 3, 1, 0],
  backlogOffset: number | null,
  paidMonths: { "YYYY-MM": true },
  active: boolean,
  createdAt: timestamp,
  updatedAt: timestamp,
  emailUpdatedAt: ISO string (from bill pipeline),
  account: string (payment method, optional),
  category: string (optional),
  subCategory: string (optional)
}
```

## Known Categories
Gold, Food (Dinner, Lunch, Supper, Snack), Groceries, Fuel (VCY52, VDA52), Kids (Allowance, School, Tuition, Clothings), Wife (Allowance, Coffee, Lunch, Shopping), Family, Loan (Auto Estima, Auto eMas, Home BTHO), Utilities (Electricity, Phone), Technology (AI, Subscription), Health & Wellness (Dental Claim), Shopping (Clothing, Electronics, Tools), Automobile (Car Maintenance, Car Wash, Insurance Roadtax), Islamic (Qurban), Rental, Salary (IPC, PCSB), Investment (Stocks Dividend), Travel (Food, Fuel, Theme park)

## FCM Notifications
- SPENT Android APK registers FCM tokens at `fcmTokens/{uid}/`
- APK class: `SpentFCMService`, `NotificationInterface`
- PWA RTDB listener on expenses path detects partner-submitted pending items
- Bill-due notifications: server-side script checks bills, sends FCM
- Channels: `spent_monitor`, `spent_reviews`, `fcm_fallback_notification_channel`

## Key Gotchas
- **Two-repo deploy**: NEVER confuse source repo vs deploy repo. Source pushes to `SPENT-staging` branch; deploy repo pushes `main` for GitHub Pages.
- **`billMonthKey`** always returns the calendar month (fixed premature-advance bug Jun 2026)
- **Auto-advance**: bills paid ahead (negative backlog) auto-advance when month flips in `renderBills()`
- **Auto-expense on paid**: Only fires from unpaid→paid, requires `account` + `category` + `amount` all set
- **Un-toggling paid does NOT delete auto-created expense**
- **Cache buster eval** at `/opt/data/evals/spent_cache_buster_eval.py` — run before every deploy (threshold 85%)
