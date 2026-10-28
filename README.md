# HackerRank Orchestrate - October 2026
## Ghost Data Still Exists: AI Deletion Audit Solution

An evidence-driven, deterministic AI auditing system that independently verifies whether customer personal data has been completely and defensibly deleted across primary databases, secondary replicas, search indexes, analytics platforms, backups, integration flows, and customer support channels.

---

## 1. System Architecture

The solution is architected as a modular, deterministic data-forensics and auditing pipeline:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Deletion Requests Input                         │
│                     (dataset/deletion_requests.csv)                    │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                     1. Intent Classification Layer                     │
│    Detects access / rectification / non-deletion requests ->           │
│    Assigns NOT_APPLICABLE with empty steps ([]) per specification      │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ (Deletion requests)
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                     2. Identity Resolution Engine                      │
│                      (code/identity_resolver.py)                       │
│    Resolves current identities & historical change logs                │
│    (user_identities.csv + user_audit_log.csv).                         │
│    Links email aliases, phone changes, names, and hashes to user_id.   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   3. Evidence Extraction & Indexing                    │
│                       (code/evidence_parser.py)                        │
│    Ingests all 94 evidence files across 5 categories:                  │
│    • SQL Snapshot Databases (primary, read, failover, regional)        │
│    • JSON Bundles & Exports (service_resource_export, snapshots)       │
│    • Structured & Unstructured Logs (traced via correlation IDs)       │
│    • PDF Documents & PNG Images (PyPDF + RapidOCR cache)               │
│    • Correlates system_activity.csv deletion worker executions         │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                  4. Temporal State & Policy Evaluator                  │
│                      (code/deletion_auditor.py)                        │
│    • Reconstructs state of all discovered resources as of audit_at     │
│    • Applies deletion_rules.csv policy rules:                          │
│      - USER_PROFILE / SEARCH_INDEX / USER_EXPORT -> DELETE             │
│      - SUPPORT_MESSAGE / ACCOUNT_TOMBSTONE -> REDACT                   │
│      - ANALYTICS_PROFILE -> ANONYMIZE                                  │
│      - DATA_FLOW -> DISABLE (active flow detection)                    │
│      - BACKUP_ARCHIVE -> BLOCK_RESTORE (restore risk detection)        │
│      - SECURITY_AUDIT_LOG -> RETAIN (legal hold / compliance)          │
│    • Outputs COMPLETE or INCOMPLETE status                             │
│    • Enforces exact phase & priority sorting:                          │
│      FINDING (with evidence_id) -> REMEDIATION -> VERIFICATION         │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                          Deliverables Output                           │
│     • output.csv (root repository level, 518 sorted cases)             │
│     • code/evaluation/evaluation_report.md                             │
│     • code/evaluation/usage_report.md                                  │
│     • code.zip                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Directory Structure

```text
├── README.md                           # Documentation and execution instructions
├── requirements.txt                    # Python runtime dependencies
├── output.csv                          # Grader-ready predictions (518 rows)
├── code.zip                            # Packaged submission archive
├── log.txt                             # Verbatim session audit trace
├── code/
│   ├── main.py                         # CLI entrypoint for running audits
│   ├── identity_resolver.py            # Historical identity resolution module
│   ├── evidence_parser.py              # Ingestion engine for 94 evidence files
│   ├── deletion_auditor.py             # Policy and temporal state evaluation engine
│   └── evaluation/
│       ├── main.py                     # Evaluation harness
│       ├── evaluation_report.md        # Comprehensive evaluation report
│       └── usage_report.md             # Execution performance and cost report
└── dataset/                            # Input dataset and evidence repository
```

---

## 3. Quick Start & Execution

### Prerequisites
- Python 3.10+ (tested on Python 3.13)
- `pip install -r requirements.txt`

### Generate Full Predictions (`output.csv`)
Run the solution on the full evaluation dataset (`dataset/deletion_requests.csv`):

```bash
python code/main.py
```

This generates `output.csv` at the repository root with exactly 518 data rows sorted by `case_id`.

### Run Custom / Sample Audits
To audit a specific request file or output to a custom path:

```bash
python code/main.py --requests dataset/sample_deletion_requests.csv --output sample_output_test.csv
```

### Run Evaluation Harness
To evaluate predictions against ground truth labels:

```bash
python code/evaluation/main.py --prediction output.csv --ground_truth dataset/sample_output.csv
```

This computes case-level, step-level, field-level, and sequence metrics, and updates `code/evaluation/evaluation_report.md`.

---

## 4. Key Guarantees & Constraints

1. **Zero Hardcoding**: All decisions, resource associations, and evidence citations are dynamically derived from evidence files and audit logs. Zero case IDs or synthetic answers are hardcoded.
2. **Deterministic & Offline**: 100% deterministic local runtime with zero paid LLM token calls, zero API keys, and $0.00 external cost.
3. **Strict Sorting Compliance**: All steps strictly follow the mandatory contract:
   - Phase Order: `FINDING` $\to$ `REMEDIATION` $\to$ `VERIFICATION`.
   - Operation Priority: Follows the official order defined in the specification.
   - Tie-breaking: Sorted alphabetically by `system_id`, then `resource_id`.
   - Step Numbering: Consecutive 1-based integer indexing.
4. **Citation Discipline**:
   - `COMPLETE` cases contain only `VERIFICATION` checks citing verified evidence.
   - `INCOMPLETE` cases cite evidence for `FINDING` steps, while `REMEDIATION` and `VERIFICATION` steps have `evidence_id: ""` as required by specification.
