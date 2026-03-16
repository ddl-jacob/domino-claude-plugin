---
name: domino-governance
description: Manage model risk governance in Domino using policies, bundles, and evidence. Covers creating governance bundles, attaching model artifacts and MLflow results as evidence, progressing through policy stages, and documenting findings. Use when the user mentions governance, compliance, bundles, policies, model risk management, SR 11-7, NIST AI RMF, or audit trails.
---

# Domino Governance Skill

This skill provides knowledge for managing model risk governance in Domino Data Lab using the Governance API.

## Configuration

The Domino Governance API base path is: `$DOMINO_API_HOST/api/governance/v1`

Some deployments expose governance on a different host than the internal API. If so, set `DOMINO_GOVERNANCE_HOST` to override the base URL. Check `domino_project_settings.md` for project-specific overrides.

The API key is available as the `$DOMINO_USER_API_KEY` environment variable.

## Key Concepts

### Policy (Template)
A **policy** is a reusable governance template that defines the stages, evidence requirements, and approval gates a model must pass through. Examples: SR 11-7, NIST AI RMF, internal model risk frameworks. Policies are created by administrators in the Domino UI; you discover them via `list_governance_policies`.

### Bundle (Living Document)
A **bundle** is the compliance document for a *specific model* in a *specific project*. It follows a policy and accumulates evidence as the model progresses through development, validation, and approval. One project can have multiple bundles (e.g., one per model version).

### Evidence (Proof)
Evidence comes in two forms:
1. **Attachments** — Files, model versions, and reports attached to the bundle (visible in the "Attachments" tab)
2. **EvidenceSet answers** — Responses to policy-defined form questions (visible in the "Evidence" tab for each stage)

### Finding (Issue)
A **finding** documents a problem, risk, or concern discovered during review. Findings have severity levels and are tracked as part of the audit trail.

## Related Documentation

- [BUNDLE-LIFECYCLE.md](./BUNDLE-LIFECYCLE.md) - Stage sequences, creating bundles, progressing stages
- [EVIDENCE-WORKFLOW.md](./EVIDENCE-WORKFLOW.md) - Attachment types, evidence submission, findings

## MCP Tools Available

| Tool | Purpose | Limitations |
|------|---------|-------------|
| `list_governance_policies` | Discover available policy templates | Works correctly |
| `create_governance_bundle` | Create a bundle for a model, attach a policy | Works correctly |
| `get_governance_bundle` | Inspect bundle: stages, attachments, status | Works — but does NOT return evidenceSet IDs |
| `list_governance_bundles` | List bundles in a project | Works correctly |
| `add_bundle_attachment` | Attach evidence | **BROKEN** — use `curl` instead (see Step 5) |
| `submit_evidence_result` | Answer policy evidence questions | **BROKEN** — use `curl` instead (see Step 6) |
| `update_bundle_stage` | Progress through policy stages | Works correctly |
| `create_governance_finding` | Document issues during review | **BROKEN** — use `curl` instead (see Step 8) |

## Standard 8-Step Governance Workflow

Follow these steps when setting up governance for a model:

### Step 1: Discover Policies
```
list_governance_policies
```
Review available templates. Note the `id` of the policy you want to use.

### Step 2: Get the Project ID
The project ID is needed to create a bundle. Use the `DOMINO_PROJECT_ID` environment variable (available inside Domino) or look it up via the gateway API.

### Step 3: Create a Bundle
```
create_governance_bundle(
    project_id="...",
    name="My Model v1.0",
    description="Description of the model being governed",
    policy_id="..."
)
```
Save the returned `bundle_id`.

### Step 4: Inspect the Bundle
```
get_governance_bundle(bundle_id="...")
```
This reveals the policy's stage structure, attachments, and approval status. **Note**: This does NOT return evidenceSet IDs — see Step 6 for how to discover those.

### Step 5: Attach Evidence
**IMPORTANT**: The `add_bundle_attachment` MCP tool does not work correctly — the API requires `identifier` as a JSON object and only supports types `ModelVersion` and `Report` (not "File" or "ExternalLink"). Use direct `curl` calls instead. See [EVIDENCE-WORKFLOW.md](./EVIDENCE-WORKFLOW.md) for the correct API format.

```bash
# Attach a registered model version
curl -X POST "$BASE/bundles/$BUNDLE_ID/attachments" \
  -H "X-Domino-Api-Key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"type":"ModelVersion","identifier":{"name":"model-name","version":5},"name":"Display Name"}'

# Attach a project file (notebook, report, etc.)
curl -X POST "$BASE/bundles/$BUNDLE_ID/attachments" \
  -H "X-Domino-Api-Key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"type":"Report","identifier":{"branch":"main","commit":"abc...","source":"git","filename":"path/to/file"},"name":"Display Name"}'
```

### Step 6: Answer Evidence Questions (EvidenceSet)

Evidence questions are the interactive forms shown in the Domino UI under each stage's "Evidence" tab. They are defined in the policy YAML as `evidenceSet` items.

**IMPORTANT**: The `submit_evidence_result` MCP tool uses incorrect field names and endpoint. Use direct `curl` calls instead.

#### 6a. Discover EvidenceSet IDs

EvidenceSet IDs are NOT in the bundle response. Fetch them from the **policy** endpoint:

```bash
curl -s "$BASE/policies/$POLICY_ID" \
  -H "X-Domino-Api-Key: $API_KEY"
```

The response contains `stages[]` → `evidenceSet[]` → `artifacts[]` with full UUIDs for each evidence item and artifact.

#### 6b. Submit Answers

Use the `submit-result-to-policy` RPC endpoint:

```bash
curl -X POST "$BASE/rpc/submit-result-to-policy" \
  -H "X-Domino-Api-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "bundleId": "bundle-uuid",
    "policyId": "policy-uuid",
    "evidenceId": "evidence-uuid",
    "content": {
      "artifact-uuid": "value"
    }
  }'
```

**Key details**:
- Use `policyId` (NOT `policyVersionId`)
- `evidenceId` is the evidence set item UUID (from policy response)
- `content` is a map of `{artifactId: value}`
- For **radio/textinput/textarea/select**: value is a string
- For **checkbox/multiSelect**: value is an array of strings
- Submit one artifact at a time per call, or multiple artifacts in the same evidence item together

### Step 7: Progress Stages
As evidence is collected and approvals obtained:
```
update_bundle_stage(
    bundle_id="...",
    stage_id="...",
    status="Complete"
)
```

### Step 8: Document Findings (if any)
**Note**: The `create_governance_finding` MCP tool is incomplete — the API also requires `policyVersionId`, `name`, `approver` (object), and `assignee` (object). Use a direct `curl` call:

```bash
curl -X POST "$BASE/findings" \
  -H "X-Domino-Api-Key: $API_KEY" -H "Content-Type: application/json" \
  -d '{
    "bundleId": "bundle-uuid",
    "policyVersionId": "policy-version-uuid",
    "name": "Finding title",
    "title": "Finding title",
    "description": "Detailed description...",
    "severity": "High",
    "approver": {"id": "org-uuid", "name": "model-gov-org"},
    "assignee": {"id": "user-uuid", "name": "claus_murmann"}
  }'
```
Get the `policyVersionId` and user/org IDs from `get_governance_bundle` response.

## Viewing in Domino UI

After creating a bundle and attaching evidence, the bundle is visible in the Domino UI:
**Project** > **Govern** > **Bundles** > click the bundle name

The UI shows:
- **Overview** — Stage progression and classification
- **Evidence** tab (per stage) — Interactive forms for evidenceSet questions
- **Attachments** tab — Files, model versions, and reports
- **Findings** tab — Documented issues with severity
