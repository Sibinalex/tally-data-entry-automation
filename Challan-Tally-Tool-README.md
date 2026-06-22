# Challan-to-Tally Automation Tool
**Jay Kay Associates | Version 1.0**

---

## 1. Project Overview

A two-part local Windows tool that reads supplier/customer challans via AI, lets the user review and correct extracted items, then runs a bot to enter them into Tally Prime automatically — eliminating manual data entry errors.

**Monthly Cost:** ~₹200–750 for 300 challans (Claude API only)
**Internet Required:** Only for OCR step (Claude API). Everything else is offline.

---

## 2. System Architecture

```
Upload Challan (Image / PDF)
        |
Claude Vision API  →  Extract: Party, Date, Challan No., Items, Qty, Rate, HSN, GST%
        |
3-Layer Matching System
        |
Review Screen  →  User edits / confirms
        |
"Confirm & Send to Tally" button
        |
PyAutoGUI Bot  →  Tally Prime  →  Sales Invoice created ✅
```

### Part 1 — OCR & Item Extraction
- Upload challan image or PDF via web interface
- Claude Haiku API reads and extracts all structured data
- 3-layer system resolves challan item names → Tally item names
- Review screen shows extracted items — fully editable
- User confirms → data passed to Part 2

### Part 2 — Tally RPA Bot
- Python bot (PyAutoGUI) takes keyboard control of Tally Prime
- Opens new Sales Invoice, enters party name and date
- Types each item one by one — name, qty, rate
- Saves voucher on completion
- ESC key stops bot anytime

---

## 3. Item Name Matching System

Challan item names rarely match Tally item names exactly. A 3-layer system handles this.

### Layer 1 — Exception Rules (Highest Priority)
User-defined rules that override everything — even exact Tally matches.

```
Challan:    "Allen Bolt 4x25"
Tally has:  "Allen Bolt 4x25"       ← exact match but wrong brand
Rule:        always map to "TVS Allen Bolt 4x25"
```

Stored in: `data/exceptions.json` (supplier-wise)

### Layer 2 — Mapping Table
All previously confirmed challan-name → Tally-name pairs, stored per supplier.
Once confirmed, never asked again.

Stored in: `data/mappings.json` (supplier-wise)

### Layer 3 — Fuzzy + AI Match
For new/unknown items not yet in mapping table:

| Confidence | Action |
|---|---|
| > 85% | Auto-suggest (shown in yellow for review) |
| 50–85% | Low-confidence suggestion with warning |
| < 50% | Marked UNMATCHED (red) — user must map manually |

Claude API used as semantic fallback — understands `"EM 10"` = `"End Mill 10mm HSS"`

### Supplier-Wise Mapping

Same challan item can map differently per supplier:

| Supplier | Challan Name | Tally Item |
|---|---|---|
| XYZ Hardware | Allen Bolt 4x25 | TVS Allen Bolt 4x25 |
| ABC Traders | Allen Bolt 4x25 | Unbrako Allen Bolt 4x25 |
| XYZ Hardware | Drill 6.5 | Addison Drill 6.5mm HSS |

### How the System Learns

- **First challan from supplier:** fuzzy match suggests, user confirms
- **System saves:** Supplier + Challan Name → Tally Item permanently
- **Next challan from same supplier:** auto-mapped, no prompt
- **Exception rules:** saved when user overrides an exact match

```
Week 1:   ~40 manual confirmations
Week 2:   80% auto-matched
Month 2:  95%+ fully automatic
```

---

## 4. Review Screen

Nothing enters Tally without passing through this screen.

### Columns

| Column | Description |
|---|---|
| Sr. No. | Row number |
| Challan Item Name | Exactly as written on challan |
| Mapped Tally Item | Resolved Tally stock item name |
| Qty | Quantity extracted |
| Unit | pcs / nos / kg / mtr etc. |
| Rate | Rate per unit |
| HSN | HSN code if present |
| GST % | GST rate if present |
| Amount | Auto-calculated (Qty × Rate) |
| Status | ✅ Green / ⚠️ Yellow / ❌ Red |

### Status Indicators
- ✅ **Green** — Auto-matched with high confidence
- ⚠️ **Yellow** — Low confidence, review recommended
- ❌ **Red** — Unmatched, must be resolved before proceeding

### User Actions
- Edit any field inline
- Change mapped Tally item via searchable dropdown
- Delete wrong rows
- Add missing items manually
- When correcting: system asks *"Save for future?"* and *"Always override?"*
- Proceed button **disabled** until zero red items remain

---

## 5. Tally RPA Bot

### Step-by-Step Bot Sequence

```
1. Alt+Tab         → Switch focus to Tally Prime
2. V               → Vouchers
3. F8              → New Sales Invoice
4. F2              → Set Date → Tab
5. Type Party Name → Enter → confirm from ledger list
6. For each item:
      Type item name → Enter
      Type Qty       → Tab
      Type Rate      → Tab → Enter
7. Ctrl+A          → Accept and save voucher ✅
```

### Safety Features
- Bot starts **only** after user clicks "Confirm & Send to Tally"
- Warning shown before start: *"Bot will take keyboard control. Do not touch keyboard/mouse until done."*
- **ESC key** stops bot immediately at any point
- Configurable keystroke delay (default 300ms) and item delay (500ms)
- Live progress shown in app UI — item by item
- Unexpected Tally screen detected → bot pauses and alerts user

### Prerequisites
- Tally Prime must be open
- Must be on **Gateway of Tally** screen before starting
- Stock items must already exist in Tally
- Party ledger must already exist in Tally

---

## 6. Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| OCR / AI Extraction | Claude Haiku API | Read challan, extract structured data |
| Semantic Matching | Claude API (fallback) | Understand abbreviated item names |
| Fuzzy Matching | RapidFuzz (Python) | Match similar names to Tally master |
| RPA Bot | PyAutoGUI + keyboard | Keyboard automation in Tally Prime |
| Backend | Python Flask (local) | Bridge between frontend and Python |
| Frontend | HTML + CSS + JS | Upload, review screen, progress display |
| Local Storage | JSON files | Mappings, exceptions, Tally item master |
| OS | Windows | Required for Tally Prime |

---

## 7. File Structure

```
challan-tally-tool/
|
+-- app.py                  ← Flask server, all API routes
+-- ocr.py                  ← Claude Vision API integration
+-- matcher.py              ← 3-layer matching logic
+-- bot.py                  ← PyAutoGUI Tally bot
+-- config.py               ← API key, delay settings
|
+-- data/
|   +-- exceptions.json     ← Override rules (supplier-wise)
|   +-- mappings.json       ← Confirmed past matches (supplier-wise)
|   +-- tally_master.json   ← Full Tally stock item list
|   +-- history/            ← Past processed challans (log)
|
+-- frontend/
|   +-- index.html          ← Upload + Review screen
|
+-- requirements.txt
```

---

## 8. Local Data Files

### tally_master.json
Full list of all stock items from Tally Prime. Reference dictionary for all matching.

**How to populate:**
1. Tally Prime → Stock Summary → Export as CSV
2. Upload CSV via app → auto-converts to `tally_master.json`
3. Refresh whenever new stock items are added in Tally

### mappings.json
```json
{
  "XYZ Hardware": {
    "Allen Bolt 4x25": "TVS Allen Bolt 4x25",
    "Drill 6.5": "Addison Drill 6.5mm HSS"
  },
  "ABC Traders": {
    "Allen Bolt 4x25": "Unbrako Allen Bolt 4x25"
  }
}
```

### exceptions.json
```json
{
  "XYZ Hardware": {
    "Allen Bolt 4x25": "TVS Allen Bolt 4x25"
  }
}
```
Exception rules always override even exact Tally matches. Editable anytime via **Manage Rules** screen in the app.

---

## 9. Implementation Steps

### Step 1 — Setup
1. Install Python 3.10+ on Windows PC
2. Run: `pip install flask pyautogui rapidfuzz keyboard anthropic pillow`
3. Get Anthropic API key from [console.anthropic.com](https://console.anthropic.com)
4. Add API key to `config.py`

### Step 2 — Load Tally Item Master
1. Tally Prime → Stock Summary → Export as CSV
2. Upload CSV via app → auto-converts to `tally_master.json`
3. Repeat whenever new stock items are added in Tally

### Step 3 — Run the App
1. Run: `python app.py`
2. Open browser: `http://localhost:5000`
3. Upload challan → review → confirm → bot enters in Tally

### Step 4 — First Challan (Training Phase)
1. Upload first challan from a supplier
2. Correct any wrong item mappings on review screen
3. Save mappings when prompted
4. Set exception rules for items that should always override
5. By Week 2 the system handles most items automatically

---

## 10. Out of Scope (Phase 1)

- Handwritten challan support (Phase 2)
- Batch processing of multiple challans at once
- Auto-creation of new party ledgers in Tally
- Auto-creation of new stock items in Tally
- Tally XML API integration (bot approach used instead)
- Purchase invoice entry (Sales Invoice only in Phase 1)

---

## 11. Summary

| Feature | Detail |
|---|---|
| Challan input | Image (JPG/PNG) or PDF |
| AI Model | Claude Haiku (fast, cost-effective) |
| Matching | 3-layer: Exception Rules + Mapping Table + Fuzzy/AI |
| Supplier awareness | Yes — mappings are supplier-specific |
| User review | Mandatory before any Tally entry |
| Bot type | Keyboard automation (PyAutoGUI) |
| Tally version | Tally Prime |
| Internet needed | Only for Claude API (OCR step) |
| Monthly cost | ~₹200–750 for 300 challans |
| Learning | Improves automatically with every challan processed |

---

*Jay Kay Associates — Internal Tool Documentation*
