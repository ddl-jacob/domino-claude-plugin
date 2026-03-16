# Evidence Workflow

This document explains how to attach evidence, answer policy questions, and create findings in governance bundles.

## Setup

The API base URL uses `$DOMINO_API_HOST` (or `$DOMINO_GOVERNANCE_HOST` if governance is on a separate host):
```bash
API_KEY="$DOMINO_USER_API_KEY"
BASE="${DOMINO_GOVERNANCE_HOST:-$DOMINO_API_HOST}/api/governance/v1"
```

## Attachment Types (API Reality)

**IMPORTANT**: The `add_bundle_attachment` MCP tool has limitations. For full control, use direct API calls via `curl` to `POST /api/governance/v1/bundles/{bundleId}/attachments`.

The API supports **two attachment types**:

### ModelVersion

Attaches a registered model version from the Domino Model Registry.

**Identifier format**: JSON object `{"name": "model-name", "version": N}`

```bash
curl -X POST "$BASE/bundles/$BUNDLE_ID/attachments" \
  -H "X-Domino-Api-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "ModelVersion",
    "identifier": {"name": "my-model", "version": 5},
    "name": "My Model v5",
    "description": "Model description and metrics."
  }'
```

**When to use**: When the model is registered in Domino's Model Registry. Use the model name and version number (integer).

### Report

Attaches a file from the project repository (git or DFS).

**Identifier format**: JSON object with branch, commit, source, and filename.

```bash
# For git-based projects:
curl -X POST "$BASE/bundles/$BUNDLE_ID/attachments" \
  -H "X-Domino-Api-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "Report",
    "identifier": {
      "branch": "main",
      "commit": "abc123def...",
      "source": "git",
      "filename": "notebooks/02_train.ipynb"
    },
    "name": "02 Model Training",
    "description": "Training notebook with MLflow logging."
  }'

# For DFS-based projects:
# Use "source": "DFS" instead of "git"
```

**When to use**: For any project file — notebooks, scripts, plots, reports, config files. The filename is the path relative to the project root. Get the commit hash with `git rev-parse HEAD`.

### What Does NOT Work

The MCP tool `add_bundle_attachment` documentation claims support for "File" and "ExternalLink" types, but **these are not valid API types**. The only valid types are:
- `ModelVersion` — with `identifier: {"name": "...", "version": N}`
- `Report` — with `identifier: {"branch": "...", "commit": "...", "source": "git"|"DFS", "filename": "..."}`

The MCP tool also sends `identifier` as a flat string, but the API requires it as a **JSON object**. Therefore, **use direct `curl` calls** for attachments.

---

## Answering Evidence Questions (EvidenceSet)

Policies define evidence questions via `evidenceSet` items in the YAML. These appear as interactive forms in the Domino UI under each stage's "Evidence" tab.

### Discovering EvidenceSet IDs

**IMPORTANT**: The `get_governance_bundle` response does NOT contain evidenceSet IDs. You must fetch them from the **policy** endpoint:

```bash
curl -s "$BASE/policies/$POLICY_ID" \
  -H "X-Domino-Api-Key: $API_KEY"
```

The response structure is:
```
stages[] → evidenceSet[] → artifacts[]
```

Each `evidenceSet` entry has:
- `id` — The evidence UUID (used as `evidenceId` in submission)
- `name` — Display name
- `artifacts[]` — Individual form fields, each with:
  - `id` — The artifact UUID (used as the key in `content`)
  - `inputType` — Form type (radio, textinput, textarea, select, checkbox, etc.)

### Submitting Answers

**IMPORTANT**: The `submit_evidence_result` MCP tool uses incorrect field names. Use direct `curl` calls with the `submit-result-to-policy` RPC endpoint:

```bash
curl -X POST "$BASE/rpc/submit-result-to-policy" \
  -H "X-Domino-Api-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "bundleId": "bundle-uuid",
    "policyId": "policy-uuid",
    "evidenceId": "evidence-set-uuid",
    "content": {
      "artifact-uuid": "value"
    }
  }'
```

### Field Reference

| Field | Description | Where to Find |
|-------|-------------|---------------|
| `bundleId` | The bundle ID | From `create_governance_bundle` or `get_governance_bundle` |
| `policyId` | The policy ID (**NOT** `policyVersionId`) | From `get_governance_bundle` → `policyId` field |
| `evidenceId` | The evidenceSet item UUID | From `GET /policies/{policyId}` → `stages[].evidenceSet[].id` |
| `content` | Map of `{artifactId: value}` | Keys from `stages[].evidenceSet[].artifacts[].id` |

### Content Value Types

| Input Type | Value Format | Example |
|------------|-------------|---------|
| `radio` | String matching an option label | `"Yes"` |
| `textinput` | Plain text string | `"Claus Murmann"` |
| `textarea` | Multi-line text string | `"Detailed description..."` |
| `select` | String matching an option label | `"Tier 2 — High"` |
| `checkbox` | Array of selected option labels | `["Claims prediction", "Reserving"]` |
| `multiSelect` | Array of selected option labels | `["Option A", "Option B"]` |
| `date` | Date string | `"2026-03-04"` |
| `numeric` | Number as string | `"0.628"` |

### Submission Tips

- You can submit one artifact at a time or multiple artifacts for the same evidenceId in one call
- Each successful submission returns the full bundle JSON
- If a policy has `classification` rules tied to an artifact, the bundle's `classificationValue` updates automatically
- The `policyId` field is the policy UUID, NOT the `policyVersionId` — this is a common gotcha

### MCP Tool vs Correct API

| MCP `submit_evidence_result` | Correct API (`submit-result-to-policy`) |
|-----|-----|
| `bundleId` | `bundleId` |
| `evidenceQuestionId` ❌ | `evidenceId` ✅ |
| `resultValue` ❌ | `content: {artifactId: value}` ✅ |
| Missing | `policyId` ✅ |

---

## Creating Findings

Findings document issues discovered during review. The MCP tool `create_governance_finding` is also incomplete — the API requires additional fields.

### Required Fields for Findings API

```bash
curl -X POST "$BASE/findings" \
  -H "X-Domino-Api-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "bundleId": "bundle-uuid",
    "policyVersionId": "policy-version-uuid",
    "name": "Finding title",
    "title": "Finding title",
    "description": "Detailed description of the issue...",
    "severity": "High",
    "approver": {"id": "org-or-user-uuid", "name": "org-name"},
    "assignee": {"id": "user-uuid", "name": "username"}
  }'
```

**Required fields the MCP tool misses**: `policyVersionId`, `name`, `approver` (as object), `assignee` (as object).

Get the `policyVersionId` from the bundle response (`get_governance_bundle`). Get user/org IDs from the `stageApprovals` section of the bundle response.

### Severity Levels

| Severity | When to Use | Example |
|----------|-------------|---------|
| Low | Minor concern, no action needed | "Training data has 2% missing values in canton field" |
| Medium | Should be addressed, not blocking | "High correlation between age and premium features" |
| High | Must be addressed before deployment | "Model AUC below policy threshold (0.628 < 0.75)" |
| Critical | Blocks deployment, immediate action | "Data leakage detected — claim_date used as feature" |

---

## Helper: Batch Attach Files

```bash
# Attach multiple project files in one go
API_KEY="$DOMINO_USER_API_KEY"
BASE="${DOMINO_GOVERNANCE_HOST:-$DOMINO_API_HOST}/api/governance/v1"
BUNDLE="your-bundle-id"
COMMIT=$(git rev-parse HEAD)
BRANCH=$(git rev-parse --abbrev-ref HEAD)

for file in "notebooks/01_eda.ipynb" "notebooks/02_train.ipynb" "notebooks/03_validate.ipynb"; do
  name=$(basename "$file")
  curl -s -X POST "$BASE/bundles/$BUNDLE/attachments" \
    -H "X-Domino-Api-Key: $API_KEY" \
    -H "Content-Type: application/json" \
    -d "{
      \"type\": \"Report\",
      \"identifier\": {\"branch\": \"$BRANCH\", \"commit\": \"$COMMIT\", \"source\": \"git\", \"filename\": \"$file\"},
      \"name\": \"$name\",
      \"description\": \"Attached from project repository.\"
    }"
done
```
