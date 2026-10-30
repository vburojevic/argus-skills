---
name: reading-medical-records
version: 1.3.3
description: Use when adding a person's medical document (lab report, discharge letter, imaging report, prescription, vaccination record) to Argus, checking its reading, recording it after the person agrees, deleting it, or reading their lab results — one history per test — document facts and the clinical records they shared from Apple Health — always stating what a record says, never interpreting it.
license: MIT
---

# Read and record medical records with argus

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

A person's medical records reach Argus two ways: **documents** they bring (a PDF, a scan or a photo of a lab report, a discharge letter, a prescription), and **health records** their providers sent to Apple Health. Both are the person's record. You read the results and facts; you never see, open or receive a document's file.

## Documents: add, review, record

```sh
argus documents add ./blood-2026-03.pdf --kind lab_report --output json
argus documents get doc_abc123 --output json        # poll until to_review or failed
argus documents confirm doc_abc123 --confirm        # only after the person agrees
argus documents list --status to_review --output json
argus documents delete doc_abc123 --results keep --confirm
```

MCP tools: `add_document`, `list_documents`, `get_document`, `confirm_document`, `delete_document`.

- **What goes in:** a PDF, JPEG, PNG or WebP the person gave you, at most 20 MB and 40 pages (an image at most 5 MB). HEIC is refused: ask for a JPEG or PDF. A password-protected PDF is refused.
- **Setup comes first.** `records_setup_required` means the person has not set up Records yet; tell them to open Records in the Argus app on their iPhone, iPad or Mac, which makes their document key and shows their recovery key once. You cannot do this for them.
- **When it cannot be read.** `document_reader_unavailable` (503) means no reader is configured right now, and `argus documents list` says `reading_available: false`; nothing was stored, so say so and try later.
- **Who reads it.** The upload answers with `reader`: the provider and model that reads the document and `keeps`, what it keeps (at launch: "Read by OpenAI, which may keep it up to 30 days for abuse monitoring and never trains on it."; Claude, once Argus reads with it, keeps nothing). Tell the person in those words before or as you add it.
- **The original is sealed.** It is encrypted to the person's own devices the moment it arrives; the server cannot open it after the read. Only the person, in an Argus app, opens it, with their passkey or Face ID. There is no command or tool for the file, and an agent key is refused (403) by the route the apps use.
- **Scans are checked too.** A PDF's figures are held to its own text layer. For a scan or a photo, Argus's apps send the text the device recognized on it (`recognized_text`, one string per page), and the reading's numbers are held to that instead of only to the reader's own quotation. You never send it: an agent has no device recognition to offer.
- **Reading takes a moment.** A new document is `reading`; poll `get_document` until it is `to_review` or `failed`. A `failed` document carries `failure.reason`; quote it. `figures_not_on_page` means the reader wrote numbers that are not printed on the page twice in a row, so nothing was kept.

### Review before anything is recorded

A `to_review` document carries `proposed.results` and `proposed.facts`, exactly as the reader copied them, each with its `page` and the printed line (`source_text`). Nothing is in the ledger yet.

- Show the person what was read, in the lab's own words, units and decimal marks ("Glukoza 5,2 mmol/L, range 3,9–5,8").
- A result with `needs_match: true` has an unsure test match: offer its `loinc_candidates` and let the person pick one, or "no match" (`loinc: null`). Never pick for them.
- A result without `collected_at` had no collection date on the page: ask for it.
- `named_patient` is the patient the document names; if it is not the person, say so before recording.
- **Record only when the person agrees in this conversation.** Without `--rows`, `argus documents confirm <id> --confirm` files the reading as read and is refused with `match_required` while any match is unsure. To file the person's edits, write them to a JSON file and pass `--rows`:

```json
{
  "results": [
    { "name": "Glukoza", "value": "5,2", "unit": "mmol/L", "range_low": "3,9", "range_high": "5,8",
      "collected_at": "2026-03-12T11:00:00.000Z", "page": 1, "loinc": "2345-7" }
  ],
  "facts": []
}
```

Without `--confirm` the command exits 4 and records nothing. Never change a value the person did not change.

### A lost key

If the person lost every device, their iCloud Keychain and their recovery key, their originals cannot be opened by anyone; their results and facts stay. To start Records again they delete each original with `--results keep` and then reset the key from an Argus app with their account password. You cannot reset it, and you never ask for their password.

### Deleting

Ask the person which they want, every time:
- `--results keep` deletes only the sealed original of a recorded document; its results and facts stay, sourced from a document they deleted.
- `--results delete` deletes the document, its original, its results and its facts.

## Lab results and facts

A recorded document carries `recorded.results` — one row per test: `value_as_written` (and `value` when it is a plain number), `unit`, `range_low`/`range_high`/`range_text` as printed, `flag_as_written` (the lab's own mark) and `flag` (`above`, `below` or `within`, from the lab's own range; absent when the lab printed none) — and `recorded.facts`: diagnoses, findings, medicines prescribed, vaccines given, procedures and allergies, in the document's own words. A key that has no value is left out, never sent as null.

### One history per test

```sh
argus labs list --output json                      # every test, newest latest result first
argus labs list --panel heart_lipids --output json # one panel
argus labs get 13457-7 --output json               # one test's history, results and documents
argus labs get "name:Glukoza natašte" --output json
```

MCP tools: `list_lab_tests` and `get_lab_history`.

- **A test is its LOINC code**; a result recorded with no match is its own series under the name as printed, keyed `name:<name>`. Pass a test's `test` key from the list to `get`.
- **Documents and Apple Health feed one history.** A Health Records lab result with a LOINC code and a value joins its test's history, kept from its record and rewritten whenever the record changes. The same test, collection day and value from two rows reads as one point (`result_ids` and `sources` name both); neither row is deleted. `results` lists every row with its `source`; `documents` lists the documents behind them, `original_deleted` when the person deleted a file and kept its results.
- **Units.** A test reads in the unit of its latest result, as the lab wrote it: a point's `value`, `range_low` and `range_high` are in that unit (older results converted through a curated UCUM table, with molar masses where mg/dL meets mmol/L). A point with `charted: false` has a unit that does not convert, or a text result: quote its `value_as_written` and `unit` and never compare it on the same scale.
- **Flags** come only from the lab's own range for that result: `above`, `below` or `within`, absent without a range. Argus never brings a population range.
- Each test carries its `panel` (heart_lipids, blood, glucose, liver, kidney, electrolytes, thyroid, iron_vitamins, inflammation, hormones) where the curated map knows its code; the Body page marks the panel's system with the test's latest result.
- A new result emits `lab_result.recorded` (see the `reacting-to-events` skill).
- A goal on a test is `lab:<loinc>`, always ongoing, scored on the latest result within 365 days (see the `goals-and-progress` skill).

## Health records from Apple Health

A health record is what a person's provider sent to Apple Health and the person then shared with Argus: a FHIR resource (allergy, clinical note, condition, coverage, immunization, lab result, medication, procedure, vital sign) or a CDA document. Argus keeps each one as received — the FHIR resource whole, the document's XML whole — beside its display name, its date and the provider it came from. The date is the provider's: when the result was taken or the condition recorded, read from the FHIR resource; never the day the record reached the phone, which stands in only when the resource names none.

```sh
argus records list --output json                       # newest first, every kind
argus records list --kind lab_result --output json     # one kind
argus records list --cursor <next_cursor> --output json
argus records get hrec_abc123 --output json            # one record whole
```

MCP tools: `get_health_records` (the list, by kind, limit and cursor) and `get_health_record` (one record by its id).

Follow each page's `next_cursor` until it is null when the question needs every record. A record's `resource` is the provider's FHIR JSON: read its own fields — `code.text` or a coding's `display`, `status`, `valueQuantity` with its `unit`, `effectiveDateTime`, `onsetDateTime`. A `cda_document` carries `document.title`, its author and custodian, and the XML.

The person shares records from Argus on iPhone, on Apple Health's own sheet. An empty list means nothing was shared; it never proves a person has no conditions, allergies or results. Apple offers Health Records only in some countries (the US, the UK, Canada and a few more); elsewhere, such as Croatia, documents are the way records reach Argus. Say that the records cannot come through Apple Health there, not that there are none.

## How to speak about records

Argus is a ledger, not a clinician:
- State what a record says, in its own words and units, with its date and provider. "Your LDL on 12 March 2026 was 4,2 mmol/L, above the lab's range of under 3,0 (Laboratorij Primjer)."
- A flag is the lab's range, never Argus's judgement: never call a result good or bad beyond what the lab printed.
- Never diagnose, never infer a condition from a record's absence, and never give medical advice or suggest a treatment.
- Say when a field is missing rather than filling it, and never write a figure the document does not print.

Skills version 1.3.3 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
