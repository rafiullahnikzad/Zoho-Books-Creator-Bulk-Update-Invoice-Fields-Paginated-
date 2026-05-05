# 🔄 Zoho Books → Creator — Bulk Update Invoice Fields (Paginated)

**Script Type:** Zoho Creator — Standalone Deluge Function (Bulk Scheduler / Manual Paginated Run)  
**Portal:** `your-portal`  
**App:** `your-app`  
**Form:** `Your_Form_Name`  
**Organization ID:** `YOUR_ORG_ID`  
**Author:** Rafiullah Nikzad — Senior Zoho Developer  
**Last Updated:** May 2026

---

## 📋 Overview

This script performs a **paginated bulk update** across all project records in Zoho Creator, syncing invoice-related fields from Zoho Books Sales Orders. It processes **200 records per run** and is designed to be executed repeatedly — incrementing `startRow` by 200 each time — until all records are covered.

The core business rule is:

| Project Status | Action |
|----------------|--------|
| `"Invoiced"` | **FILL** — fetch SO from Books, populate all budget/invoice fields |
| Anything else | **CLEAR** — wipe all budget/invoice fields to empty |

This ensures that only legitimately invoiced projects carry financial data, and all others are clean.

---

## 🎯 Purpose

Use this script to:

- Backfill `Invoiced_date`, `Total_Invoice_amount`, `Engine`, `Location`, and `Invoice_Status` for thousands of Creator project records
- Automatically clear those fields for any project not in `"Invoiced"` status
- Run as a manual paginated job across large record sets in batches of 200
- Serve as a one-time historical data fix or a periodic reconciliation job

---

## ⚙️ Key Configuration

```javascript
organizationId = "YOUR_ORG_ID";   // ← Zoho Books Org ID
startRow = 1;                      // ← CHANGE each run (+200 per batch)
```

### Pagination Schedule

Each run processes rows `startRow` to `startRow + 199`. Increment by **200** every run:

| Run | startRow | Covers Rows |
|-----|----------|-------------|
| 1 | `1` | 1 – 200 |
| 2 | `201` | 201 – 400 |
| 3 | `401` | 401 – 600 |
| 4 | `601` | 601 – 800 |
| … | … | … |
| N | `(N-1)*200 + 1` | until no records returned |

> The script automatically prints `▶ NEXT RUN startRow = XXXX` at the end of every execution so you always know the next value to set.

---

## 🔄 Script Flow

```
START
  │
  ├─► Fetch 200 Creator records from startRow
  │       └── [no records returned] → Log "No more records" → EXIT
  │
  ├─► For each record:
  │       │
  │       ├─► Read: ID, Project_Name, Status, Sales_order_no
  │       │
  │       ├─► [Status != "Invoiced"] ──► CASE A: CLEAR
  │       │       └── updateRecord with all fields set to ""
  │       │             ├── ✅ clearedCount++
  │       │             └── ❌ errors++
  │       │
  │       └─► [Status == "Invoiced"] ──► CASE B: FILL
  │               │
  │               ├─► [soNumber empty] → SKIP (skipped++)
  │               │
  │               ├─► GET /salesorders?salesorder_number={soNumber}
  │               │       └── [not found] → SKIP (skipped++)
  │               │
  │               ├─► GET /salesorders/{soId} (full detail)
  │               │       └── [API error] → errors++, continue
  │               │
  │               ├─► Parse custom_fields → engineVal, locationVal
  │               │
  │               ├─► Parse invoices[0].status → map to Creator dropdown
  │               │
  │               ├─► Convert date: yyyy-MM-dd → dd-MMM-yyyy
  │               │
  │               └─► updateRecord (only non-empty fields)
  │                       ├── ✅ successCount++
  │                       └── ❌ errors++
  │
  └─► Print summary: Filled | Cleared | Skipped | Errors | Next startRow

END
```

---

## 📡 API Calls Per Record

| Call | Method | Endpoint |
|------|--------|----------|
| Fetch Creator records | `zoho.creator.getRecords` | Your form, 200 at a time |
| Search SO by number | `GET` | `/books/v3/salesorders?salesorder_number={soNumber}` |
| Fetch SO full detail | `GET` | `/books/v3/salesorders/{salesorder_id}` |
| Update Creator record | `zoho.creator.updateRecord` | Your form |

**Books Connection:** `zoho_books_connection`  
**Creator Connection:** `zoho_oauth_connection`

> ⚠️ **API Rate Limit Awareness:** Each Invoiced record triggers 2 Books API calls + 1 Creator update. For 200 records with ~50% Invoiced, that's ~150 API calls per run. Stay within Zoho's rate limits (typically 100 req/min for Books).

---

## 🗂️ Fields Updated Per Record

### CASE A — Status ≠ `"Invoiced"` (CLEAR)

| Creator Field | Value Set |
|---------------|-----------|
| `Invoiced_date` | `""` (empty) |
| `Total_Invoice_amount` | `""` (empty) |
| `Engine` | `""` (empty) |
| `Location` | `""` (empty) |
| `Invoice_Status` | `""` (empty) |

### CASE B — Status = `"Invoiced"` (FILL from Books SO)

| Creator Field | Source |
|---------------|--------|
| `Invoiced_date` | SO `date` → converted to `dd-MMM-yyyy` |
| `Total_Invoice_amount` | SO `total` |
| `Engine` | SO custom field `cf_engine` (`value_formatted`) |
| `Location` | SO custom field `cf_location` (`value_formatted`) |
| `Invoice_Status` | First linked invoice `status` → mapped to dropdown label |

---

## 🗂️ Invoice Status Mapping

| Books Value | Creator Dropdown |
|-------------|-----------------|
| `draft` | `Draft` |
| `sent` | `Sent` |
| `paid` | `Paid` |
| `overdue` | `Overdue` |
| `void` | `Void` |
| *(other)* | Passed through as-is |

> ⚠️ Verify your Creator `Invoice_Status` dropdown labels match exactly.

---

## 📅 Date Conversion

```javascript
invoicedDate = soDate.toDate("yyyy-MM-dd").toString("dd-MMM-yyyy");
```

Books returns `2024-11-15` → Creator stores `15-Nov-2024`. If `soDate` is empty or `"null"`, `invoicedDate` is excluded from the `updateMap` to avoid overwriting with blank.

---

## 📊 Run Summary Output

At the end of each run, the script prints a full summary:

```
══════════════════════════════════════════
startRow  : 201 → 400
✅ Filled : 87
🧹 Cleared: 94
⏭ Skipped : 12
❌ Errors  : 7
▶ NEXT RUN startRow = 401
══════════════════════════════════════════
```

---

## 🔐 Prerequisites

| Requirement | Details |
|-------------|---------|
| Books OAuth Connection | `zoho_books_connection` with `ZohoBooks.salesorders.READ` scope |
| Creator OAuth Connection | `zoho_oauth_connection` with read + write access to your Creator app |
| `Sales_order_no` field | Must be pre-populated in Creator records for FILL case to work |
| `Status` field | Must contain exactly `"Invoiced"` (case-sensitive) for FILL to trigger |
| Creator field names | `Invoiced_date`, `Total_Invoice_amount`, `Engine`, `Location`, `Invoice_Status` must match your form's API field names exactly |

---

## 🧪 How to Run

1. Open **Zoho Creator** → **Settings → Functions**
2. Create or open the standalone function
3. Replace all placeholders: `YOUR_ORG_ID`, `your-portal`, `your-app`, `Your_Form_Name`
4. Set `startRow = 1` for the first run
5. Click **Test** to execute
6. Note the `▶ NEXT RUN startRow` value at the bottom of the log
7. Update `startRow` and repeat until:
   ```
   ✅ No more records at startRow: XXXX
   ```

---

## 🐞 Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `⏭ SKIP — No SO number` | `Sales_order_no` field is empty in Creator | Run the SO number extraction script first |
| `⏭ SKIP — SO not found in Books` | SO number doesn't match Books exactly | Check leading zeros; Books may store `17100` vs `017100` |
| `Invoice_Status` not updating | Dropdown label mismatch | Verify Creator dropdown choices match the mapped values |
| `❌ SO detail failed` | Books API returned non-zero code | Check connection scopes and org ID |
| All records show `🧹 Cleared` | `Status` value doesn't match `"Invoiced"` exactly | Check for trailing spaces or case differences |
| Script stops mid-run | Zoho execution time limit hit | Reduce batch to 100 records |

---

## 📈 Performance Estimates

| Total Records | Batch Size | Estimated Runs | Est. Time per Run |
|---------------|------------|----------------|-------------------|
| 5,000 | 200 | ~25 runs | 2–5 minutes |
| 8,700 | 200 | ~44 runs | 2–5 minutes |
| 8,700 | 100 | ~87 runs | 1–3 minutes |

---

## 🔗 Relationship to Other Scripts

| Script | Scope | When to Use |
|--------|-------|-------------|
| **This script** | All records, paginated | Full bulk backfill or reconciliation |
| `single_record_fix` | 1 specific record | Emergency fix for a single record |
| `debug_so_invoice` | Read-only diagnosis | When a record isn't updating correctly |
| `extract_sales_order_no` | Populate `Sales_order_no` | Run this first if SO numbers are missing |
| `books_to_creator_invoice_sync` | Ongoing new records | Webhook/workflow trigger for new syncs |

**Recommended execution order for a fresh backfill:**
```
1. extract_sales_order_no        ← Populate Sales_order_no from Project_Name
2. bulk_update_invoice_fields    ← THIS SCRIPT — fill/clear all budget fields
3. books_to_creator_invoice_sync ← Keep ongoing records in sync going forward
```

---

## 📄 Full Script

```javascript
// ============================================================
// BULK UPDATE — FINAL VERSION
// Rule: Project Status == "Invoiced" → FILL budget from Books SO
//       Project Status != "Invoiced" → CLEAR all budget fields
// ⚙ Change startRow each run: 1, 201, 401, 601 ...
// Replace: YOUR_ORG_ID, your-portal, your-app, Your_Form_Name
// ============================================================
organizationId = "YOUR_ORG_ID";
startRow = 1; // ← CHANGE EACH RUN (+200)

successCount = 0;
clearedCount = 0;
skipped = 0;
errors = 0;

allRecs = zoho.creator.getRecords("your-portal","your-app","Your_Form_Name","",startRow,200,"zoho_oauth_connection");
creatorData = allRecs.get("data");

if(creatorData == null || creatorData.size() == 0)
{
    info "✅ No more records at startRow: " + startRow;
    return;
}
info "▶ Processing " + creatorData.size() + " records | startRow: " + startRow;

for each rec in creatorData
{
    creatorRecId  = rec.get("ID");
    projectName   = rec.get("Project_Name") + "";
    projectStatus = rec.get("Status") + "";
    soNumber      = rec.get("Sales_order_no") + "";

    info "── " + projectName + " | Project Status: [" + projectStatus + "]";

    // ============================================================
    // CASE A — NOT Invoiced → CLEAR all budget fields
    // ============================================================
    if(projectStatus != "Invoiced")
    {
        clearMap = Map();
        clearMap.put("Invoiced_date","");
        clearMap.put("Total_Invoice_amount","");
        clearMap.put("Engine","");
        clearMap.put("Location","");
        clearMap.put("Invoice_Status","");

        clearResp = zoho.creator.updateRecord("your-portal","your-app","Your_Form_Name",creatorRecId.toLong(),clearMap,Map(),"zoho_oauth_connection");
        if(clearResp.get("code") == 3000)
        {
            info "   🧹 Cleared";
            clearedCount = clearedCount + 1;
        }
        else
        {
            info "   ❌ Clear FAILED: " + clearResp;
            errors = errors + 1;
        }
        continue;
    }

    // ============================================================
    // CASE B — IS Invoiced → FILL from Books SO
    // ============================================================
    if(soNumber == "" || soNumber == "null")
    {
        info "   ⏭ SKIP — No SO number";
        skipped = skipped + 1;
        continue;
    }

    // ── Fetch SO summary ──────────────────────────────────────
    soSearchResp = invokeurl
    [
        url :"https://www.zohoapis.com/books/v3/salesorders?organization_id=" + organizationId + "&salesorder_number=" + soNumber
        type :GET
        connection:"zoho_books_connection"
    ];
    soList = soSearchResp.get("salesorders");

    if(soList == null || soList.size() == 0)
    {
        info "   ⏭ SKIP — SO not found in Books: " + soNumber;
        skipped = skipped + 1;
        continue;
    }

    soSummary = soList.get(0);
    soId      = soSummary.get("salesorder_id") + "";
    soDate    = ifnull(soSummary.get("date"),"") + "";
    soTotal   = ifnull(soSummary.get("total"),0);

    // ── Fetch full SO detail for Engine, Location & Invoice ───
    soDetailResp = invokeurl
    [
        url :"https://www.zohoapis.com/books/v3/salesorders/" + soId + "?organization_id=" + organizationId
        type :GET
        connection:"zoho_books_connection"
    ];
    if(soDetailResp.get("code") != 0)
    {
        info "   ❌ SO detail failed: " + soDetailResp;
        errors = errors + 1;
        continue;
    }
    soFull      = soDetailResp.get("salesorder");
    engineVal   = "";
    locationVal = "";
    invStatus   = "";

    // ── Engine & Location from SO custom fields ───────────────
    soCfList = soFull.get("custom_fields");
    if(soCfList != null)
    {
        for each cf in soCfList
        {
            apiName = ifnull(cf.get("api_name"),"") + "";
            if(apiName == "cf_engine")
            {
                engineVal = ifnull(cf.get("value_formatted"),"") + "";
            }
            else if(apiName == "cf_location")
            {
                locationVal = ifnull(cf.get("value_formatted"),"") + "";
            }
        }
    }

    // ── Invoice Status from linked invoice ────────────────────
    invoicesArr = soFull.get("invoices");
    if(invoicesArr != null && invoicesArr.size() > 0)
    {
        invStatusRaw = ifnull(invoicesArr.get(0).get("status"),"") + "";
        if(invStatusRaw == "draft")        { invStatus = "Draft"; }
        else if(invStatusRaw == "sent")    { invStatus = "Sent"; }
        else if(invStatusRaw == "paid")    { invStatus = "Paid"; }
        else if(invStatusRaw == "overdue") { invStatus = "Overdue"; }
        else if(invStatusRaw == "void")    { invStatus = "Void"; }
        else if(invStatusRaw != "")        { invStatus = invStatusRaw; }
    }

    // ── Convert date ──────────────────────────────────────────
    invoicedDate = "";
    if(soDate != "" && soDate != "null")
    {
        invoicedDate = soDate.toDate("yyyy-MM-dd").toString("dd-MMM-yyyy");
    }

    // ── Update Creator ────────────────────────────────────────
    updateMap = Map();
    if(invoicedDate != "")  { updateMap.put("Invoiced_date",invoicedDate); }
    updateMap.put("Total_Invoice_amount",soTotal);
    if(engineVal != "")     { updateMap.put("Engine",engineVal); }
    if(locationVal != "")   { updateMap.put("Location",locationVal); }
    if(invStatus != "")     { updateMap.put("Invoice_Status",invStatus); }

    updateResp = zoho.creator.updateRecord("your-portal","your-app","Your_Form_Name",creatorRecId.toLong(),updateMap,Map(),"zoho_oauth_connection");
    if(updateResp.get("code") == 3000)
    {
        info "   ✅ " + invoicedDate + " | $" + soTotal + " | " + invStatus + " | " + engineVal;
        successCount = successCount + 1;
    }
    else
    {
        info "   ❌ FAILED: " + updateResp;
        errors = errors + 1;
    }
}

info "══════════════════════════════════════════";
info "startRow  : " + startRow + " → " + (startRow + 199);
info "✅ Filled : " + successCount;
info "🧹 Cleared: " + clearedCount;
info "⏭ Skipped : " + skipped;
info "❌ Errors  : " + errors;
info "▶ NEXT RUN startRow = " + (startRow + 200);
info "══════════════════════════════════════════";
```

---

## 👤 Author

**Rafiullah Nikzad**  
Senior Zoho Developer | Zoho Authorized Partner  
🌐 [rafiullahnikzad.netlify.app](https://rafiullahnikzad.netlify.app)  
💻 [github.com/rafiullahnikzad](https://github.com/rafiullahnikzad)  
🔗 LinkedIn: Zoho Afghanistan Community
