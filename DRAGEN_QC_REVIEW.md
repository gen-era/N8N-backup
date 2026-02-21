# DRAGEN QC Flow — Review Report

**Date:** 2026-02-21
**Branch:** `claude/check-dragen-qc-flow-bN0ac`
**Scope:** `DRAGEN QC.json` (main, manual) and `Sub - DRAGEN QC.json` (subworkflow, active)

---

## Summary

Two workflows handle DRAGEN QC processing:

| File | Status | Trigger | Nodes |
|------|--------|---------|-------|
| `DRAGEN QC.json` | **inactive** | Manual | 44 |
| `Sub - DRAGEN QC.json` | **active** | Called by another workflow | 26 |

Both run DRAGEN v4.3.13 alignment + QC coverage analysis against reference `hg38_3`, collect mapping/coverage metrics, and write them to PostgreSQL (`measurement_reads` table).

---

## Issues Found

### CRITICAL

#### 1. Sub-workflow silently drops errors (Sub - DRAGEN QC.json)
**Node:** `Is Error`
**Connection:** `Is Error` → output[0] (error branch) → `[]` (empty — nothing)

When DRAGEN fails (stderr contains `ERROR`, `syntax error`, `fatal error`, `Caught signal Aborted`, or `Please run sosreport`), the error branch connects to nothing. No DB update occurs, no notification is sent, and the status in `data_operations` remains `24` (pending) forever.

**Compare:** The main `DRAGEN QC.json` correctly routes errors to `Error Data OP - CES`, which updates `status_concept_ref` to `27` and writes `stderr` to `error_text`.

**Fix needed:** Add an error-logging node in the sub-workflow's `Is Error` true branch, equivalent to `Error Data OP - CES` in the main workflow.

---

#### 2. Sub-workflow trigger input declaration is incomplete (Sub - DRAGEN QC.json)
**Node:** `When Executed by Another Workflow`
**Declared inputs:** `[{ name: "run_id" }]`
**Actually used:** `conversion_ref` is referenced throughout the workflow (in `Select rows from a table`, `Get Conversion`)

The `conversion_ref` parameter is not declared in the trigger's `workflowInputs` block, which means callers have no schema-level guarantee it will be provided. If a caller omits `conversion_ref`, all downstream DB queries silently get `undefined`.

**Fix needed:** Add `{ name: "conversion_ref" }` to the trigger's `workflowInputs`.

---

### HIGH

#### 3. Hardcoded run ID in main workflow (DRAGEN QC.json)
**Node:** `Set Run ID`
**Value:** `"251226_A01832R_0234_BHJJFLDRX7"` (hardcoded string)

This is a development/test artifact. The main workflow is manually triggered and processes whichever run ID is set here. Anyone running it without updating this value will silently reprocess the same historical run.

**Fix needed:** Either prompt for the run ID at trigger time (via a form trigger or workflow input), or remove this node if the workflow is no longer in use.

---

#### 4. Find Conversion Path produces comma-separated output used as shell path arguments (Sub - DRAGEN QC.json)
**Node:** `Find Conversion Path`
**Command:**
```bash
fdfind "{{ $json.id_source_value }}" /mnt/smb_share/Conversions/ /mnt/illuminastorage/fastq_folder/ /mnt/sapiens_dragen/tmp/ --max-depth 1 | paste -sd ','
```

The `paste -sd ','` joins multiple matching paths with commas. The result (`path1,path2,...`) is then interpolated directly into the `Find Fastq Files` command:
```bash
fdfind "${sample_id}_" ${stdout}
```

The shell does not split on commas, so `fdfind` receives the entire comma-joined string as a single argument — not as multiple directory paths. This works only when exactly one path is found.

**Fix needed:** Replace `paste -sd ','` with a newline separator, or use `head -n 1` if only the first result is needed. Alternatively, in `Find Fastq Files`, split on commas before using as arguments.

---

### MEDIUM

#### 5. No duplicate data-operation guard in sub-workflow (Sub - DRAGEN QC.json)
**Missing node:** Equivalent of `Check OP Record - CES` + `Is Data OP new?`

The main workflow checks whether a `data_operations` row already exists (with `recipe_ref = 6`) before inserting a new one, and skips the sample if it has already been processed successfully (status ≠ 27). The sub-workflow always inserts a new `data_operations` row unconditionally via `Log Data Op`, which can create duplicate records if the sub-workflow is re-triggered for the same conversion.

---

#### 6. Test pin data left in active sub-workflow (Sub - DRAGEN QC.json)
**pinData:**
```json
"When Executed by Another Workflow": [{ "json": { "conversion_ref": 300 } }]
```

Pinned data overrides real inputs when executing in the n8n UI's manual test mode, which may cause confusion during debugging or re-testing.

---

### LOW

#### 7. String replace only swaps first occurrence of mount path (Sub - DRAGEN QC.json)
**Node:** `Add R1 and R2 Read Paths`
**Expression:**
```js
$json.stdout.split('\n').map(l => l.trim()).find(l => l.includes('_R1_')).replace('smb_share', 'koza_alldata')
```

JavaScript's `.replace(string, string)` replaces only the first occurrence. If a FastQ path ever contains the literal string `smb_share` more than once (e.g., `smb_share/smb_share/...`), only the first will be replaced. This is unlikely in practice but fragile.

**Fix (low priority):** Use a regex with the global flag: `.replace(/smb_share/g, 'koza_alldata')`.

---

#### 8. Bed mapping: main workflow maps by kit name string; sub-workflow maps by kit_ref integer
This is a design inconsistency. The main workflow relies on the `Library_Prep_Kit` column in the SampleSheet (or DB), while the sub-workflow relies on `kit_ref` integers (1, 40, 2, 73, 97). Both cover the same 5 BED files, but:

- The main workflow has **7 entries** (including case-variant duplicates `Sophia-CES-v3` / `SOPHIA-CES-V3` and separator variants `WES-mtDNA` / `WES+mtDNA`).
- The sub-workflow has **5 entries** (no duplicates needed since integer keys are exact).

No missing mappings were found, but the two approaches should be documented to avoid future drift.

---

## Flow Summary (Sub - DRAGEN QC)

```
When Executed by Another Workflow (conversion_ref, run_id)
  └─► Check DRAGEN Availability (SSH ps aux | grep dragen | wc -l)
        └─► Is DRAGEN Idle? (stdout == "0")
              YES ─► Get Conversion (SELECT FROM conversions WHERE uniqueref = conversion_ref)
                       └─► Find Conversion Path (fdfind id_source_value in mount points)
                             └─► Select rows from a table (SELECT FROM reads WHERE conversion_ref)
                                   ├─► Bed Mapping1 (static kit_ref → BED file map)
                                   └─► Add Bed Path (merge by kit_ref)
                                         └─► Custom Filter (sample_project regex)
                                               └─► Filter (bed_file_path not empty)
                                                     └─► Loop Over Items
                                                           └─► Log Data Op (INSERT data_operations, status=24)
                                                                 └─► Find Fastq Files (fdfind sample_id in conversion path)
                                                                       └─► Add R1 and R2 Read Paths (extract R1/R2 paths)
                                                                             └─► DRAGEN QC (SSH: run /opt/dragen/4.3.13/bin/dragen)
                                                                                   └─► Is Error?
                                                                                         NO  ─► Collect Metrics - CES
                                                                                         │         ├─► Get Concepts - CES → Merge Concepts - CES
                                                                                         │         └─► JSON Parse - CES ──────────────────────►
                                                                                         │                                                     └─► Record Measurements - CES
                                                                                         │                                                           └─► Complete Data Op - CES (status=29)
                                                                                         │                                                                 └─► DRAGEN QC Cleanup (rm -rf output/*)
                                                                                         │                                                                       └─► Loop Over Items (next sample)
                                                                                         YES ─► [NOTHING — BUG: no error logging]
              NO  ─► [stop]
```

---

## Recommendations

| Priority | Action |
|----------|--------|
| Critical | Add error-logging node to `Sub - DRAGEN QC` `Is Error` true branch |
| Critical | Declare `conversion_ref` in sub-workflow trigger inputs |
| High | Remove hardcoded run ID from main workflow `Set Run ID` node |
| High | Fix `Find Conversion Path` → `Find Fastq Files` path passing (comma vs space) |
| Medium | Add duplicate data-op guard to sub-workflow |
| Medium | Remove pinned test data from active sub-workflow |
| Low | Use global regex replace for mount path substitution |
