---
name: reading-health-records
version: 1.0.1
description: Use when listing or reading the clinical health records and CDA documents a person shared with Argus from Apple Health — allergies, conditions, immunizations, lab results, medications, procedures, vital signs, clinical notes, coverage — or explaining what a record says without interpreting it.
license: MIT
---

# Read health records with argus

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

## What a health record is

A health record is what a person's provider sent to Apple Health and the person then shared with Argus: a FHIR resource (allergy, clinical note, condition, coverage, immunization, lab result, medication, procedure, vital sign) or a CDA document. Argus keeps each one as received — the FHIR resource whole, the document's XML whole — beside its display name, its date and the provider it came from. The date is the provider's: when the result was taken or the condition recorded, read from the FHIR resource. It is never the day the record reached the phone; that day stands in only when the resource names none. Records are not samples: they never enter goals, trends, relationships or the Brief.

The person shares records from Argus on iPhone (Account → Health records), on Apple Health's own sheet. An empty list means nothing was shared; it never proves a person has no conditions, allergies or results. Apple offers Health Records only in some countries (the US, the UK, Canada and a few more). Elsewhere Apple Health holds no clinical records at all, so say that the records cannot come through Apple Health there, not that there are none.

## List and read

```sh
argus records list --output json                       # newest first, every kind
argus records list --kind lab_result --output json     # one kind
argus records list --cursor <next_cursor> --output json
argus records get hrec_abc123 --output json            # one record whole
```

MCP tools: `get_health_records` (the list, by kind, limit and cursor) and `get_health_record` (one record by its id).

Follow each page's `next_cursor` until it is null when the question needs every record.

A record's `resource` is the provider's FHIR JSON: read its own fields — `code.text` or a coding's `display`, `status`, `valueQuantity` with its `unit`, `effectiveDateTime`, `onsetDateTime`. A `cda_document` carries `document.title`, its author and custodian, and the XML.

## How to speak about records

Argus is a ledger, not a clinician:
- State what a record says, in its own words and units, with its date and provider. "Your LDL on 2 March 2026 was 3.1 mmol/L (St. Mary's Lab)."
- Never diagnose, never read a result as good or bad unless the record itself carries a reference range or interpretation — then quote it as the provider's.
- Never infer a condition from a record's absence, and never give medical advice or suggest a treatment.
- Say when a field is missing rather than filling it.

Skills version 1.0.1 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
