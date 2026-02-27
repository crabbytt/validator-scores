# validator-scores

Shared score bucket for **TrajectoryRL (SN11)** validator consensus. Validators publish per-UID evaluation scores here as signed JSON files, enabling deterministic stake-weighted consensus across all validators.

Reference: [INCENTIVE_MECHANISM.md § Validator Consensus](https://github.com/trajectoryRL/trajectoryRL/blob/main/INCENTIVE_MECHANISM.md#validator-consensus)

---

## How It Works

```
Validator evaluates miner packs via ClawBench
  → Signs score payload with sr25519
  → Pushes to personal fork
  → Opens PR against this repo
  → CI verifies signature + schema → auto-merges
  → All validators pull merged scores → compute stake-weighted consensus
```

Each epoch, validators independently evaluate miner PolicyBundles using ClawBench, then publish their raw `final_score` per UID. Other validators pull these published scores and compute a **stake-weighted mean** to arrive at a deterministic consensus — same data, same computation, same winner.

---

## Repository Structure

```
validator-scores/
├── epoch-42/
│   ├── 5F3sa...hotkey_A.json     # scores from validator A
│   ├── 5Gw2p...hotkey_B.json     # scores from validator B
│   └── ...
├── epoch-43/
│   └── ...
└── .github/
    └── workflows/
        └── verify-scores.yml     # CI: signature + schema validation, auto-merge
```

Score files are organized by epoch, with each file named after the validator's ss58 hotkey address.

---

## Score File Schema

```json
{
  "validator_hotkey": "5F3sa...",
  "epoch": 42,
  "block_height": 302400,
  "scores": {
    "uid_0": {
      "final_score": 0.87,
      "per_scenario": {
        "client_escalation": 0.92,
        "morning_brief": 0.85,
        "inbox_to_action": 0.88,
        "team_standup": 0.83
      }
    },
    "uid_1": {
      "final_score": 0.91,
      "per_scenario": { "...": "..." }
    }
  },
  "signature": "0x3a1b..."
}
```

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `validator_hotkey` | string | Yes | Validator's ss58 hotkey address |
| `epoch` | int | Yes | Epoch number (`current_block // blocks_per_epoch`) |
| `block_height` | int | Yes | Block height at time of evaluation |
| `scores` | dict | Yes | `uid_str` → `{final_score, per_scenario}` |
| `signature` | string | Yes | sr25519 signature over the canonical payload |

### Signature

The `signature` field is an sr25519 signature over the canonical JSON payload (all fields **except** `signature`, serialized with `sort_keys=True` and no extra whitespace). Git commit signing is not used — sr25519 is incompatible with git's GPG/SSH signing, so payload-level signing is the authentication mechanism.

---

## Submission Flow (for Validators)

### Prerequisites

- `gh` CLI installed and authenticated (`gh auth login`)
- Fork of this repo under your GitHub account
- Bittensor wallet registered on SN11
- Environment configured (see below)

### Environment Variables

| Variable | Description |
|----------|-------------|
| `WALLET_NAME` | Bittensor wallet name |
| `WALLET_HOTKEY` | Bittensor hotkey name |
| `GITHUB_TOKEN` | GitHub personal access token (push to fork) |
| `VALIDATOR_SCORES_FORK_URL` | Your fork URL (e.g. `https://github.com/you/validator-scores.git`) |
| `GITHUB_EMAIL` | Git commit email |
| `GITHUB_NAME` | Git commit name |

### Automated Publishing

The `ScorePublisher` class in `trajectoryrl.utils.score_publisher` handles the full flow automatically:

```python
from trajectoryrl.utils.config import ValidatorConfig
from trajectoryrl.utils.score_publisher import ScorePublisher

config = ValidatorConfig.from_env()

publisher = ScorePublisher(
    wallet_name=config.wallet_name,
    wallet_hotkey=config.wallet_hotkey,
    fork_repo_url=config.validator_scores_fork_url,
    local_path=config.validator_scores_local_path,
    github_token=config.github_token,
    git_email=config.git_email,
    git_name=config.git_name,
)

# Publish scores (sign → commit → push → open PR)
await publisher.publish_scores(
    epoch=42,
    block_height=150000,
    scores={
        "uid_0": {"final_score": 0.85, "per_scenario": {"client_escalation": 0.92}},
        "uid_1": {"final_score": 0.72, "per_scenario": {"client_escalation": 0.72}},
    },
)

# Pull all validators' scores and compute consensus
all_files = await publisher.pull_all_scores(epoch=42)
consensus = ScorePublisher.compute_consensus(all_files, metagraph)
```

### Manual Steps (what `ScorePublisher` does under the hood)

1. **Fork** this repo (one-time setup)
2. **Create** `epoch-{N}/{hotkey}.json` with sr25519 signature
3. **Push** to a branch on your fork (`scores/epoch-{N}-{hotkey}`)
4. **Open PR** against `trajectoryRL/validator-scores`
5. CI verifies → auto-merges

---

## CI Verification Pipeline

Every PR is automatically validated before merge:

1. PR adds/updates a **single file** matching `epoch-{N}/{hotkey}.json`
2. JSON passes **schema validation** (required fields present and typed correctly)
3. `signature` is a **valid sr25519 signature** over the payload, signed by the `validator_hotkey`
4. `validator_hotkey` matches a **registered validator** in the current metagraph
5. Validator has **non-zero stake**

If all checks pass → **auto-merge**. If any check fails → PR rejected with a comment.

---

## Consensus Computation

All validators pull from the same `main` branch and compute:

```
consensus_score[uid] = Σ(stake_i × score_i[uid]) / Σ(stake_i)
```

Where `stake_i` is validator *i*'s TAO stake from the metagraph.

The winner is the miner with the highest `consensus_score`, subject to the δ first-mover threshold (0.05) and earliest on-chain commitment block for tie-breaking.

### Convergence

Validators operate asynchronously — no synchronized phases or deadlines:

```
Hour 0.0:  Epoch starts
Hour 0.5:  Val A finishes eval → publishes → sets weights (own scores only)
Hour 1.0:  Val B publishes → all re-pull → re-compute → set_weights
Hour 1.5:  Val C publishes → all converge on same consensus → set_weights
Hour 2-24: All validators re-submit converged weights every tempo (~72 min)
```

---

## Verification (Consumer Side)

Before including a score file in aggregation, validators independently verify:

1. `validator_hotkey` matches a registered validator in the current metagraph
2. `signature` is valid sr25519 over the payload
3. Validator has non-zero stake (rejects deregistered validators)

These checks are **redundant with CI** (defense-in-depth). Even if CI is compromised, each validator rejects invalid files locally.

### Deduplication

If a validator pushes multiple score files for the same epoch (e.g., restart mid-epoch), only the file with the highest `block_height` is used. This prevents double-counting stake in the weighted denominator.

---

## Resilience

This repo is a **coordination layer**, not the source of truth. On-chain weights (`set_weights`) are immutable and determine emissions.

| Scenario | Recovery |
|----------|----------|
| Repo lost/corrupted | Reconstruct from any validator's fork (git is distributed) |
| Scores missing | Validators re-run ClawBench — scoring is deterministic (regex-based) |
| CI compromised | Every validator independently rejects invalid signatures on pull |
| Validator offline | Others converge without it; Yuma consensus handles partial participation |

---

## Security Properties

| Property | Mechanism |
|----------|-----------|
| **No direct push** | Validators submit via PR only; CI bot merges |
| **Tamper-proof** | sr25519 payload signatures — can't forge without private key |
| **Audit trail** | Every submission is a PR, visible in merge history |
| **Deterministic** | All validators read same `main` branch → same consensus |
| **Sybil-resistant** | Stake-weighted mean — zero-stake validators are excluded |

---

## Related

- [INCENTIVE_MECHANISM.md](https://github.com/trajectoryRL/trajectoryRL/blob/main/INCENTIVE_MECHANISM.md) — Full scoring formula, winner selection, anti-gaming measures
- [VALIDATOR_OPERATIONS.md](https://github.com/trajectoryRL/trajectoryRL/blob/main/VALIDATOR_OPERATIONS.md) — Validator setup, cost projections, Docker deployment
