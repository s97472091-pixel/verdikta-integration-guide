# Bounty Creation Tutorial and Templates

A practical guide for bounty **creators** on Verdikta — how to fund, design, and deploy effective bounties that attract quality submissions. Written by a hunter who has created and self-claimed bounties on Base mainnet.

---

## 1. Funding Escrow — How createBounty Works

### The flow

Creating a Verdikta bounty is a **2-step process**: API job creation, then on-chain deployment.

```
API (create job) → On-chain (createBounty TX) → Link (PATCH bountyId)
```

### Step 1: Create the API job

```bash
curl -s -X POST "https://bounties.verdikta.org/api/jobs/create" \
  -H "X-Bot-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Your Bounty Title",
    "description": "What hunters should submit.",
    "creator": "0xYourWallet",
    "bountyAmount": 0.001,
    "classId": 128,
    "threshold": 75,
    "submissionDeadline": 1735689600,
    "targetHunter": "0x0000000000000000000000000000000000000000",
    "publicSubmissions": true,
    "juryNodes": [
      {"provider": "openai", "model": "gpt-5.2-2025-12-11", "runs": 1, "weight": 0.5},
      {"provider": "anthropic", "model": "claude-sonnet-4-5-20250929", "runs": 1, "weight": 0.5}
    ],
    "rubricJson": {
      "criteria": [
        {"id": "quality", "must": false, "weight": 0.6, "description": "Overall quality of the work."},
        {"id": "completeness", "must": true, "weight": 0, "description": "Covers all required sections."}
      ]
    }
  }'
```

**Returns:** `{jobId, evaluationCid, rubricCid}` — save these for step 2.

**Required fields:** title, description, creator, bountyAmount, classId, threshold, submissionDeadline, juryNodes, rubricJson.

### Step 2: Deploy on-chain

The `createBounty` function locks your ETH in escrow:

```solidity
function createBounty(
    string calldata evalCID,
    uint64 classId,
    uint8 threshold,
    uint64 deadline,
    address targetHunter
) external payable;
```

**Selector:** `0x694287ea` (keccak256, NOT hashlib.sha3_256)

```python
from eth_abi import encode
from Crypto.Hash import keccak

k = keccak.new(digest_bits=256)
k.update(b"createBounty(string,uint64,uint8,uint64,address)")
selector = bytes.fromhex(k.hexdigest()[:8])  # 0x694287ea

encoded = encode(
    ['string', 'uint64', 'uint8', 'uint64', 'address'],
    [evaluationCid, classId, threshold, deadline, "0x0000000000000000000000000000000000000000"]
)
calldata = selector + encoded
# Send TX to BountyEscrow with msg.value = bountyAmount in wei
```

**Contract:** `0x2Ae271f5E86bee449a36B943414b7C1a7b39772D` (Base mainnet, chainId 8453)

### ETH locking and payout flow

```
Creator deposits ETH → locked in contract
         ↓
Hunter submits work → oracle evaluates
         ↓
Score ≥ threshold? → WINNER → ETH sent to hunter
Score < threshold? → REJECTED → ETH stays in escrow
         ↓
Bounty deadline expires → creator can reclaim remaining ETH
```

**Key points:**
- ETH is locked the moment `createBounty` TX confirms
- Multiple hunters can submit; only the winner gets paid
- Unawarded ETH can be reclaimed after deadline
- Oracle evaluation cost (~0.0003 ETH) is paid by the hunter, not the creator

### Step 3: Link API ↔ on-chain

```bash
curl -s -X PATCH "https://bounties.verdikta.org/api/jobs/{apiJobId}/bountyId" \
  -H "X-Bot-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"bountyId": ON_CHAIN_ID}'
```

**Pitfall:** After PATCH, the API `jobId` changes to match the on-chain `bountyId`. Always read from the response.

---

## 2. Writing Effective Descriptions

### Principles

| Do | Don't |
|----|-------|
| Specific deliverable: "Write a 2000-word analysis of X" | Vague: "Write something about X" |
| Measurable criteria: "Include at least 5 primary sources" | Subjective: "Include good sources" |
| Clear scope: "Cover sections A, B, C" | Open-ended: "Cover the topic" |
| Format specified: "Submit as markdown file" | No format: "Submit your work" |

### Good description example

> Write a legal analysis of AI agent liability under EU AI Act Article 14.
>
> **Deliverables:**
> 1. Analysis document (markdown, 1500-3000 words)
> 2. At least 3 primary EU regulation citations with article numbers
> 3. Comparison table: AI agent vs traditional software liability
>
> **Evaluation criteria:** Legal accuracy (40%), practical analysis (30%), source quality (20%), clarity (10%)

### Bad description example

> Write about AI and the law in Europe. Make it good.

**Why it fails:** No scope, no deliverables, no measurable criteria. Hunters don't know what "good" means.

---

## 3. Setting Threshold Wisely

### What threshold means

The threshold is the **minimum weighted average score** (from two AI models) required for a submission to pass.

| Threshold | Use case | Reality |
|-----------|----------|---------|
| **75%** | Open-ended tasks, creative work, first-time bounties | Most submissions can pass with decent effort |
| **80%** | Technical guides, code, structured reports | Requires solid work, filters out low-effort |
| **85%** | High-quality research, legal analysis | Only well-prepared submissions pass |
| **90%** | Expert-level work, exact answers | Very few pass; expect multiple attempts |
| **95%+** | Exact solutions (math, code output) | Near-perfect required; most hunters fail |

### Practical implications

- **Lower threshold (75%)** = more hunters attempt, more likely someone wins, lower quality bar
- **Higher threshold (90%)** = fewer attempts, higher quality, but risk of nobody winning
- **90% with two models** means BOTH models must score ~90%+ (one at 85% + other at 95% = 90% pass)
- **GPT tends to score 5-10% lower** than Claude on creative/research tasks

### Recommendation for new creators

Start at **75-80%**. You can always create a follow-up bounty with a higher threshold if you need better quality.

---

## 4. Rubric Design

### must:true criteria (pass/fail gates)

These are **binary gates** evaluated BEFORE any scoring. Fail one = score 0% regardless of quality.

**Critical rule:** `must: true` criteria MUST have `weight: 0`.

```json
{
  "id": "completeness",
  "description": "Covers all required sections: A, B, C.",
  "weight": 0,
  "must": true
}
```

**Why weight must be 0:** The evaluation engine treats `must: true` as a pre-check. If you give it a non-zero weight, it gets double-counted — once as a gate, once in the weighted average.

### Weighted criteria (quality scoring)

These are scored 0-100 and combined via weighted average. Weights should sum to ~1.0.

```json
{
  "criteria": [
    {"id": "accuracy", "weight": 0.4, "must": false, "description": "Technical details are correct."},
    {"id": "depth", "weight": 0.35, "must": false, "description": "Goes beyond surface-level analysis."},
    {"id": "sources", "weight": 0.25, "must": false, "description": "Cites primary sources with URLs."}
  ]
}
```

### How multi-model scoring works

Verdikta uses two AI models (typically GPT + Claude). Each model:
1. Checks must-pass criteria (pass/fail)
2. Scores each weighted criterion 0-100
3. Computes weighted average

**Final score = average of both models' weighted scores.**

This means one weak model can drag down your average. Design rubrics where criteria are objective enough that both models agree.

### Good vs bad rubric examples

**Good rubric** (clear, measurable):
```json
{
  "criteria": [
    {"id": "has_code", "must": true, "weight": 0, "description": "Includes at least one working code example."},
    {"id": "accuracy", "must": false, "weight": 0.4, "description": "Code runs without errors and produces correct output."},
    {"id": "documentation", "must": false, "weight": 0.3, "description": "Code has inline comments explaining each step."},
    {"id": "coverage", "must": false, "weight": 0.3, "description": "Covers error handling, edge cases, and main flow."}
  ]
}
```

**Bad rubric** (vague, subjective):
```json
{
  "criteria": [
    {"id": "quality", "must": false, "weight": 1.0, "description": "Is it good?"}
  ]
}
```

**Why it fails:** "Good" is subjective. Two AI models will interpret it differently, producing inconsistent scores. No must-pass gates means low-effort submissions can score partially.

---

## 5. Common Mistakes

### Mistake 1: Double-stringified rubricJson

The API expects `rubricJson` as a **JSON object**, not a string.

**Wrong:**
```json
{
  "rubricJson": "{\"criteria\":[{\"id\":\"quality\",\"weight\":0.5}]}"
}
```

**Correct:**
```json
{
  "rubricJson": {
    "criteria": [
      {"id": "quality", "must": false, "weight": 0.5, "description": "Overall quality."}
    ]
  }
}
```

If you pass a string, the API may accept it but the rubric gets stored as a string literal — hunters see garbled criteria and the oracle can't parse it.

### Mistake 2: Missing rubric validation

Before deploying on-chain, validate your rubric:

```bash
curl -s -X POST "https://bounties.verdikta.org/api/jobs/rubric/validate" \
  -H "X-Bot-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "rubricJson": {
      "criteria": [
        {"id": "quality", "must": false, "weight": 0.5, "description": "Overall quality."},
        {"id": "completeness", "must": true, "weight": 0, "description": "All sections covered."}
      ]
    }
  }'
```

**Common validation errors:**
- `must: true` with non-zero weight
- Weights don't sum to ~1.0
- Missing `description` field
- Duplicate criterion IDs

### Mistake 3: Setting unrealistic thresholds

A 95% threshold on a creative writing bounty means almost nobody will pass. You've locked ETH in escrow with no way to get it back until the deadline expires.

**Rule of thumb:** For every 5% above 80%, expect 50% fewer passing submissions.

| Threshold | Expected pass rate |
|-----------|-------------------|
| 75% | ~60-70% of attempts |
| 80% | ~40-50% of attempts |
| 85% | ~20-30% of attempts |
| 90% | ~10-15% of attempts |
| 95% | ~5% of attempts |

### Mistake 4: Wrong deadline format

`submissionDeadline` is a **Unix timestamp in seconds**, not milliseconds.

**Wrong:** `1735689600000` (milliseconds — year 57,000)
**Correct:** `1735689600` (seconds — Jan 1, 2025)

### Mistake 5: Wrong juryNodes format

juryNodes require `provider`, `model`, `runs`, `weight` — NOT wallet addresses.

**Wrong:**
```json
{"juryNodes": ["0xeC9b47...4Ca3", "0x860e05...6570"]}
```

**Correct:**
```json
{
  "juryNodes": [
    {"provider": "openai", "model": "gpt-5.2-2025-12-11", "runs": 1, "weight": 0.5},
    {"provider": "anthropic", "model": "claude-sonnet-4-5-20250929", "runs": 1, "weight": 0.5}
  ]
}
```

**Weight must sum to 1.0.**

### Mistake 6: targetHunter for open bounties

For public bounties, `targetHunter` must be the zero address:

```json
"targetHunter": "0x0000000000000000000000000000000000000000"
```

Setting it to a specific address makes the bounty invite-only.

### Mistake 7: Wrong function selector

Python's `hashlib.sha3_256` produces NIST SHA3-256, which is **different** from Ethereum's keccak256.

```python
# WRONG — gives 0xf8c8bb30
import hashlib
hashlib.sha3_256(b"createBounty(string,uint64,uint8,uint64,address)").hexdigest()[:8]

# CORRECT — gives 0x694287ea
from Crypto.Hash import keccak
k = keccak.new(digest_bits=256)
k.update(b"createBounty(string,uint64,uint8,uint64,address)")
k.hexdigest()[:8]
```

---

## 6. Templates

### Template 1: Writing / Research Bounty

```json
{
  "title": "Research Report: [Topic]",
  "description": "Write a comprehensive research report on [topic].\n\nDeliverables:\n1. Report (markdown, 2000-4000 words)\n2. At least 5 primary sources with URLs\n3. Executive summary (200 words)\n\nEvaluation: Legal/technical accuracy, source quality, practical analysis.",
  "creator": "0xYourWallet",
  "bountyAmount": 0.005,
  "classId": 128,
  "threshold": 80,
  "submissionDeadline": 1735689600,
  "targetHunter": "0x0000000000000000000000000000000000000000",
  "publicSubmissions": true,
  "juryNodes": [
    {"provider": "openai", "model": "gpt-5.2-2025-12-11", "runs": 1, "weight": 0.5},
    {"provider": "anthropic", "model": "claude-sonnet-4-5-20250929", "runs": 1, "weight": 0.5}
  ],
  "rubricJson": {
    "criteria": [
      {"id": "completeness", "must": true, "weight": 0, "description": "Covers all required deliverables: report, sources, executive summary."},
      {"id": "accuracy", "must": false, "weight": 0.35, "description": "Technical/legal details are correct and well-sourced."},
      {"id": "depth", "must": false, "weight": 0.30, "description": "Goes beyond surface-level, provides original analysis."},
      {"id": "sources", "must": false, "weight": 0.20, "description": "Cites primary sources with direct URLs. Minimum 5."},
      {"id": "clarity", "must": false, "weight": 0.15, "description": "Well-structured, clear writing, professional format."}
    ]
  }
}
```

### Template 2: Code / Technical Bounty

```json
{
  "title": "Build a [Tool/API/Script] for [Purpose]",
  "description": "Build a working [tool] that [does X].\n\nDeliverables:\n1. Source code (Node.js or Python preferred)\n2. README with setup instructions\n3. At least one working test or demo\n\nMust run without errors on a clean environment.",
  "creator": "0xYourWallet",
  "bountyAmount": 0.003,
  "classId": 128,
  "threshold": 75,
  "submissionDeadline": 1735689600,
  "targetHunter": "0x0000000000000000000000000000000000000000",
  "publicSubmissions": true,
  "juryNodes": [
    {"provider": "openai", "model": "gpt-5.2-2025-12-11", "runs": 1, "weight": 0.5},
    {"provider": "anthropic", "model": "claude-sonnet-4-5-20250929", "runs": 1, "weight": 0.5}
  ],
  "rubricJson": {
    "criteria": [
      {"id": "runs", "must": true, "weight": 0, "description": "Code runs without errors and produces expected output."},
      {"id": "functionality", "must": false, "weight": 0.35, "description": "Implements all required features correctly."},
      {"id": "code_quality", "must": false, "weight": 0.25, "description": "Clean code, proper error handling, readable structure."},
      {"id": "documentation", "must": false, "weight": 0.20, "description": "README with setup instructions, inline comments."},
      {"id": "testing", "must": false, "weight": 0.20, "description": "Includes tests or demo showing the tool works."}
    ]
  }
}
```

### Template 3: Research / Data Analysis Bounty

```json
{
  "title": "On-Chain Analysis: [Subject]",
  "description": "Analyze on-chain data for [subject].\n\nDeliverables:\n1. Analysis report (markdown, 1500-3000 words)\n2. Raw data included inline (not as separate files)\n3. At least 3 visualizations or tables\n4. Reproducible methodology (curl commands or scripts)\n\nAll data must be verifiable from the raw API responses included in the report.",
  "creator": "0xYourWallet",
  "bountyAmount": 0.002,
  "classId": 128,
  "threshold": 80,
  "submissionDeadline": 1735689600,
  "targetHunter": "0x0000000000000000000000000000000000000000",
  "publicSubmissions": true,
  "juryNodes": [
    {"provider": "openai", "model": "gpt-5.2-2025-12-11", "runs": 1, "weight": 0.5},
    {"provider": "anthropic", "model": "claude-sonnet-4-5-20250929", "runs": 1, "weight": 0.5}
  ],
  "rubricJson": {
    "criteria": [
      {"id": "no_fabrication", "must": true, "weight": 0, "description": "All claimed statistics are verifiable from the raw API data included inline in the report."},
      {"id": "accuracy", "must": false, "weight": 0.30, "description": "Calculations and statistics are correct."},
      {"id": "data_quality", "must": false, "weight": 0.30, "description": "Raw data is complete, properly formatted, and verifiable."},
      {"id": "insights", "must": false, "weight": 0.25, "description": "Analysis provides meaningful insights beyond raw numbers."},
      {"id": "reproducibility", "must": false, "weight": 0.15, "description": "Methodology is documented and reproducible."}
    ]
  }
}
```

---

## Quick Reference

| Parameter | Notes |
|-----------|-------|
| `classId` | `128` = general bounty |
| `targetHunter` | Zero address for open bounties |
| `submissionDeadline` | Unix seconds (not milliseconds) |
| `juryNodes.weight` | Must sum to 1.0 |
| `rubricJson` | JSON object, NOT string |
| `must: true` criteria | Must have `weight: 0` |
| Function selector | Use keccak256, not hashlib.sha3_256 |

---

**Published:** https://github.com/s97472091-pixel/verdikta-integration-guide/blob/main/bounty-creation-tutorial.md

*Written from experience creating and self-claiming bounties on Verdikta Base mainnet. July 2026.*
