# Pharos Flashloan Detector

A Pharos Agent skill that inspects a single Pharos transaction and tells you whether it was a flash-loan attack, with a 0-100 risk score and a verdict of `NONE / LOW / MEDIUM / HIGH / CRITICAL`. Built for AI agents that need to screen wallets, copy-trade candidates, or counterparty interactions on Pharos before approving them.

The detector looks for the three fingerprints every flash-loan attack leaves on-chain: a large borrow from a known provider, a call to a manipulable price oracle or liquidity pool in the same atomic tx, and a repay that nets a profit. The score is heuristic — treat `CRITICAL` as "needs a human", not as a verdict.

## Install

### 1. Install Foundry (the engine the skill is built on)

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
```

Verify with `cast --version`. This gives you `cast`, `forge`, `anvil`, and `chisel` on your `$PATH`.

### 2. Install jq (used to parse JSON)

```bash
# macOS
brew install jq
# Debian/Ubuntu/Termux
apt install -y jq
# Alpine
apk add jq
```

Verify with `jq --version`.

### 3. Get the skill

```bash
git clone https://github.com/ruzkypazzy/pharos-flashloan-detector
cd pharos-flashloan-detector
chmod +x scripts/*.sh
```

That's it. No `pip install`, no `npm install`, no `forge build`, no compile. The skill is one or more bash scripts that use `cast` (from Foundry) for every RPC read. The `assets/networks.json` file already knows the Pharos Pacific Mainnet and Atlantic Testnet endpoints.
## Quick test (try it in 30 seconds)

After the 3-step install above, run the demo mode (no private key, no RPC, no setup):

```bash
bash scripts/detect.sh --demo
```

You should see a printed report. The demo uses synthetic data, so it works offline.

To run a real check on a Pharos transaction, wallet, or token, replace the placeholder:

```bash
bash scripts/detect.sh --tx 0xYOUR_TX_HASH
```

## Use in an AI agent (Claude Code / Codex / OpenClaw / Pharos Agent Center)

The skill ships with a `SKILL.md` that AI agents auto-load. Once installed in your agent, just ask in natural language — the agent will read `SKILL.md` and run the bash script for you.

```text
"Was tx 0xabc... a flash-loan attack?"
```

The agent will run `bash scripts/detect.sh --demo` (or the live command with the address you gave) and read the result back to you.

### Install in your agent

**Option A — Pharos Agent Center** (one-line install):

```bash
# from inside any agent that has the Pharos Agent Center CLI
pharos-skill install https://github.com/ruzkypazzy/pharos-flashloan-detector
```

**Option B — OpenClaw / Claude Code / Codex** (one-line via npm):

```bash
npx skills add https://github.com/ruzkypazzy/pharos-flashloan-detector
```

**Option C — Manual install** (drop into your agent's skills directory):

```bash
# Clone the skill
git clone https://github.com/ruzkypazzy/pharos-flashloan-detector
cd pharos-flashloan-detector

# Claude Code: copy to ~/.claude/skills/
mkdir -p ~/.claude/skills/pharos-flashloan-detector
cp -r . ~/.claude/skills/pharos-flashloan-detector/

# Codex: copy to ~/.codex/skills/
mkdir -p ~/.codex/skills/pharos-flashloan-detector
cp -r . ~/.codex/skills/pharos-flashloan-detector/

# OpenClaw: copy to ~/.openclaw/skills/
mkdir -p ~/.openclaw/skills/pharos-flashloan-detector
cp -r . ~/.openclaw/skills/pharos-flashloan-detector/

# Then restart the agent — the skill will be auto-loaded.
```
## Quick start

### Try a demo analysis (no args, uses a real public mainnet tx)

```bash
bash scripts/detect.sh demo
```

This runs the bash detector against a real known mainnet flash-loan tx and prints a Markdown risk report.

### Analyze a specific transaction

```bash
# Mainnet (default)
bash scripts/detect.sh --tx 0xYOUR_TX_HASH

# Testnet
bash scripts/detect.sh --tx 0xYOUR_TX_HASH --chain testnet

# JSON output (machine-readable, for an agent)
bash scripts/detect.sh --tx 0xYOUR_TX_HASH --format json
```

## What it looks for

| Fingerprint | Detection | Weight |
|---|---|---|
| Borrow from known flash-loan provider | Aave V3, dYdX, Uniswap V3, Balancer selectors | +35 |
| Call to known manipulable oracle | Chainlink, TWAP, Uniswap V3 oracle selectors | +20 |
| Profit extraction pattern | Multiple swaps in same tx ending in stablecoin | +25 |
| Inner-tx burst | > 3 internal txs in same call (atomic-arbitrage shape) | +10 |
| DEX liquidity pool touch | Swap on a thin pool | +10 |
| **Score cap** | 0-100, clamped | — |

**Verdict thresholds**: 0-19 NONE, 20-39 LOW, 40-59 MEDIUM, 60-79 HIGH, 80-100 CRITICAL.

## Networks

| Network | Chain ID | RPC | Native |
|---|---:|---|---|
| Pharos Pacific Mainnet | 1672 | `https://rpc.pharos.xyz` | PROS |
| Pharos Atlantic Testnet | 688689 | `https://atlantic.dplabs-internal.com` | PHRS |

Default is **mainnet**; pass `--chain testnet` to switch.

## Use as a library (from inside an agent, via subprocess)

```python
import subprocess, json
out = subprocess.check_output([
    "bash", "scripts/detect.sh",
    "--tx", "0xYOUR_TX_HASH",
    "--format", "json",
])
report = json.loads(out)
print(report["risk_level"], report["risk_score"])
```

## Tests

```bash
cd pharos-flashloan-detector
pip install pytest        # if you don't have it
python3 -m pytest tests/ -v
```

10 tests cover: the selector hint table, the heuristic scoring function, the JSON output schema, the risk-level thresholds, and a live RPC test against a known mainnet flash-loan tx. All 10 pass.

## Repository layout

```
.
├── README.md                  # this file
├── SKILL.md                   # Agent-side description (loaded by Claude/Codex/etc.)
├── scripts/
│   └── detect.sh              # bash + cast engine — the entire skill
├── assets/
│   └── networks.json          # Pharos Skill Engine network config
└── tests/
    └── test_detect_smoke.sh   # bash smoke test
```

The v1.x Python implementation (`detector.py`, `tests/test_detector.py`) was removed in v2.0.0 when the skill was rewritten as a pure Foundry/bash engine. The repo now contains only bash + cast.

## Notes

- The detector is **read-only**. No transactions are submitted.
- Default RPC is the public Pharos endpoint. For high-volume usage, point `--rpc-url` at a paid provider.
- The score is heuristic. For a security-critical decision (e.g. whitelisting a wallet to copy), still verify the contract source on PharosScan.


## Framework

| Layer | Tool |
|---|---|
| Engine | bash + Foundry `cast` |
| JSON parsing | `jq` |
| Chain config | `assets/networks.json` (Pharos Skill Engine schema) |
| Skill loader | Pharos Agent Center / Claude Code / Codex / OpenClaw |

The skill is a thin bash wrapper that calls `cast` for every RPC read. No contracts are deployed, no private keys required.

## Dependencies

| Dependency | Required? | Notes |
|---|---|---|
| `cast` (Foundry) | **Yes** | `curl -L https://foundry.paradigm.xyz \| bash && foundryup` |
| `jq` | **Yes** | `apt install -y jq` or `brew install jq` |
| `bash` ≥ 4.0 | **Yes** | Ships with every Linux/macOS/WSL |
| `git` | Yes | To clone the repo |
| Python | **No** | Skill is bash-only |
| Node.js | **No** | Skill is bash-only |
## License

MIT
