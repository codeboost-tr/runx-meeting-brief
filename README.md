# runx Meeting Brief Skill

A production-ready [runx](https://runx.ai) skill that synthesizes meeting briefs from contextual inputs — event details, provided notes, thread snippets, and public link notes. Built for **runx-cli v0.6.14**.

## What is runx?

[runx](https://runx.ai) is an open-source runtime for policy-bounded agent skills. Skills are packaged as a contract (`SKILL.md`), a machine manifest (`X.yaml`), and a runner (`run.mjs`). Every run seals a verifiable receipt.

## Skill Overview

The meeting-brief skill accepts structured inputs and generates a formatted meeting brief:

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| event_context | string | yes | JSON with title, attendees, date |
| provided_notes | string | no | Pre-written meeting notes |
| thread_snippets | string | no | Relevant thread/chat snippets |
| public_link_notes | string | no | Notes from public links |
| expected_version | string | no | Version pin |
| idempotency_key | string | no | Idempotency key for replay safety |

### Skills Are Auto-Detected

```bash
$ runx list skills --json
```

```json
{
  "schema": "runx.list.v1",
  "items": [
    {
      "kind": "skill",
      "name": "meeting-prep",
      "source": "local",
      "status": "ok",
      "harness_cases": 2
    }
  ]
}
```

```bash
$ runx skill inspect ./skills/meeting-brief --json
```

```json
{
  "name": "meeting-prep",
  "description": "A skill to synthesize a meeting brief from various contextual inputs.",
  "runners": ["default"],
  "status": "ok",
  "version": "0.1.0"
}
```

## File Structure

```
skills/meeting-brief/
├── SKILL.md       # Human-readable contract (inputs, outputs, source config)
├── X.yaml         # Machine manifest (catalog, harness cases, runner definitions)
└── run.mjs        # Node.js runner implementation
```

## How It Works

### SKILL.md — The Contract

The frontmatter declares the skill identity, inputs, and execution source:

```yaml
---
name: meeting-prep
description: A skill to synthesize a meeting brief from various contextual inputs.
source:
  type: cli-tool
  command: node
  args:
    - run.mjs
  timeout_seconds: 30
  sandbox:
    profile: readonly
    cwd_policy: skill-directory
inputs:
  event_context:
    type: string
    required: true
  provided_notes:
    type: string
  thread_snippets:
    type: string
  public_link_notes:
    type: string
runx:
  category: ops
  input_resolution:
    required:
      - event_context
---
```

### X.yaml — The Machine Manifest

Defines the catalog, harness test cases, and runner configuration:

```yaml
skill: meeting-prep
version: "0.1.0"

catalog:
  kind: skill
  audience: public
  visibility: public
  role: canonical

harness:
  cases:
    - name: sealed-meeting-brief
      runner: default
      inputs:
        event_context: '{"title":"Sync","attendees":["Alice","Bob"],"date":"2026-07-02"}'
        provided_notes: "Alice needs to discuss budget."
        thread_snippets: "Bob: I will bring the Q3 report."
        public_link_notes: "Q3 budget is $10k."
        expected_version: "1"
        idempotency_key: "abc-123"
      expect:
        status: sealed
        receipt:
          schema: runx.receipt.v1
          state: sealed
          disposition: closed

    - name: missing-inputs-brief
      runner: default
      inputs:
        event_context: '{}'
      expect:
        status: sealed
        receipt:
          schema: runx.receipt.v1
          state: sealed
          disposition: closed

runners:
  default:
    default: true
    type: cli-tool
    command: node
    args:
      - run.mjs
    inputs:
      event_context:
        type: string
        required: true
      provided_notes:
        type: string
      thread_snippets:
        type: string
      public_link_notes:
        type: string
      expected_version:
        type: string
      idempotency_key:
        type: string
```

**Key concepts:**
- `harness.cases` — inline test scenarios with `expect.status: sealed` (must complete)
- `runners.default` — defines the Node.js CLI tool runner
- `receipt` block in expect — asserts the receipt schema, state, and disposition

### run.mjs — The Runner

A Node.js script reads inputs from `RUNX_INPUT_*` environment variables, synthesizes a brief, and outputs JSON:

```javascript
const event_context_raw = process.env.RUNX_INPUT_EVENT_CONTEXT || "{}";
const provided_notes = process.env.RUNX_INPUT_PROVIDED_NOTES || "";
const thread_snippets = process.env.RUNX_INPUT_THREAD_SNIPPETS || "";
const public_link_notes = process.env.RUNX_INPUT_PUBLIC_LINK_NOTES || "";
// ... parses context, generates brief, outputs JSON
```

## Running the Harness

```bash
# Verify the skill loads without errors
runx doctor --json
```

```json
{
  "schema": "runx.doctor.v1",
  "status": "success",
  "summary": { "errors": 0, "warnings": 0, "infos": 0 }
}
```

```bash
# Run the harness (requires signing keys for production receipts)
runx harness ./skills/meeting-brief --json
```

Expected output includes sealed receipt IDs:

```
{
  "status": "passed",
  "case_count": 2,
  "receipt_ids": ["sha256:..."],
  "case_names": ["sealed-meeting-brief", "missing-inputs-brief"]
}
```

## Sealed Receipts

Every harness run produces a sealed receipt — the unit of trust in runx:

```json
{
  "schema": "runx.receipt.v1",
  "id": "sha256:052e91300884c03daae761d64d73a8ec2381a24d6035850e80b4d39807b38951",
  "issuer": { "type": "hosted", "kid": "..." },
  "signature": { "alg": "Ed25519", "value": "base64:..." },
  "subject": {
    "kind": "skill",
    "ref": { "type": "harness", "uri": "hrn_run_default_..." }
  },
  "seal": {
    "disposition": "closed",
    "reason_code": "process_closed"
  }
}
```

Receipts can be verified independently:

```bash
runx verify --receipt receipt.json --json
```

## Publishing to Registry

```bash
runx login --provider github --for publish
runx registry publish ./skills/meeting-brief
```

The hosted registry runs the harness as the publish gate — all cases must pass before the skill is published.

## Resources

- [runx GitHub](https://github.com/runxhq/runx)
- [runx.ai](https://runx.ai)
- [Frantic Board](https://gofrantic.com)
- [SKILL.md Specification](https://runx.ai/docs/skill-md)
