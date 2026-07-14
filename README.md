# Task 17 – The Relational Data Merger

An n8n workflow that ingests two independent mock data streams, deduplicates each one, merges them on a shared email key, and separates matched records from unmatched ones.

---

## What it does

Real automation pipelines rarely get clean data. Ad platforms send duplicate leads. CRMs have orphaned records. This workflow handles both before writing anything to a database.

**The pipeline has 8 nodes across 4 stages:**

```
Stream 1 (Names + Emails)     →  Dedup Stream 1  ─┐
                                                    ├→  Merge on Email  →  IF: Matched?  →  Matched Records
Stream 2 (Emails + Domains)   →  Dedup Stream 2  ─┘                                    →  Unmatched Records
```

---

## Node breakdown

### Stage 1 — Data Generation
**Stream 1 – Names & Emails**
Simulates an ad platform or lead capture form. Returns 6 records (includes one intentional duplicate to test dedup).

**Stream 2 – Emails & Domains**
Simulates a CRM or data enrichment source. Returns 7 records (includes one duplicate and one record with no matching email in Stream 1).

### Stage 2 — Deduplication *(added beyond task requirements)*
**Dedup Stream 1 / Dedup Stream 2**
Code nodes that track seen emails in a `Set` and drop any record after the first occurrence. Handles case and whitespace normalization (`toLowerCase().trim()`). Logs the before/after count to the console.

This is necessary because in real pipelines, ad platforms routinely send duplicate lead submissions. Merging before deduplication would produce duplicate rows in the output.

### Stage 3 — Merge
**Merge – Join on Email**
Uses n8n's Merge node in **Combine / Matching Fields** mode, joining on `email`. Set to `keepEverything` (outer join) so unmatched records also pass through — they get caught and flagged in the next stage rather than silently dropped.

### Stage 4 — Error Handling *(added beyond task requirements)*
**Matched or Unmatched? (IF node)**
Checks whether both `name` and `company_domain` are present. If either is missing, the email existed in only one stream — meaning the merge had no counterpart to join it with.

- **True branch → Matched Records:** Outputs clean `{ name, email, company_domain, status: "matched" }`
- **False branch → Unmatched Records:** Flags the record with `status: "unmatched"` and a `reason` field explaining which stream it came from

---

## Output structure

**Matched Records (5 items)**
```json
{
  "name": "Aarav Sharma",
  "email": "aarav@example.com",
  "company_domain": "example.com",
  "status": "matched"
}
```

**Unmatched Records (1 item)**
```json
{
  "name": null,
  "email": "ghost@unknown.org",
  "company_domain": "unknown.org",
  "status": "unmatched",
  "reason": "Email in Stream 2 only — no name found"
}
```

---

## How to run

1. Open n8n
2. Click the **+** menu → **Import from file**
3. Select `task17_relational_data_merger.json`
4. Click **Execute Workflow**

No credentials or external services required. Everything runs locally with mock data.

---

## Why these additions matter

The task asked for a merge. A merge on raw data is fine in a demo but breaks in production — duplicates cause double-inserts, and silently dropped unmatched records make pipelines look like they worked when they didn't.

The dedup nodes and error-handling branch are the difference between a workflow that demonstrates a concept and one that could actually be deployed.
