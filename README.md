# Pharos Flashloan Detector

A Pharos Agent skill that inspects a single Pharos transaction and tells you whether it was a flash-loan attack, with a 0-100 risk score and a verdict of `NONE / LOW / MEDIUM / HIGH / CRITICAL`. Built for AI agents that need to screen wallets, copy-trade candidates, or counterparty interactions on Pharos before approving them.

The detector looks for the three fingerprints every flash-loan attack leaves on-chain: a large borrow from a known provider, a call to a manipulable price oracle or liquidity pool in the same atomic tx, and a repay that nets a profit. The score is heuristic — treat `CRITICAL` as "needs a human", not as a verdict.

## Install

```bash
git clone https://github.com/ruzkypazzy/pharos-flashloan-detector
cd pharos-flashloan-detector
chmod +x scripts/detect.sh
```

No build step, no Foundry required. Pure Python 3.10+ standard library. The only optional dependency is `pytest` if you want to run the test suite.

## Quick start

### Try a demo analysis (no args, uses a real public mainnet tx)

```bash
bash scripts/detect.sh demo
```

This runs `python3 detector.py demo` against a real known mainnet flash-loan tx and prints a Markdown risk report.

### Analyze a specific transaction

```bash
# Mainnet (default)
bash scripts/detect.sh 0xYOUR_TX_HASH

# Testnet
bash scripts/detect.sh 0xYOUR_TX_HASH --chain testnet

# JSON output (machine-readable, for an agent)
bash scripts/detect.sh 0xYOUR_TX_HASH --json
```

You can also call the Python module directly:

```bash
python3 detector.py --tx 0xYOUR_TX_HASH
python3 detector.py --tx 0xYOUR_TX_HASH --chain mainnet --json
```

### Use as a Python library (from inside an agent)

```python
from detector import FlashloanDetector

det = FlashloanDetector(rpc="https://rpc.pharos.xyz", chain="mainnet")
report = det.analyze("0xYOUR_TX_HASH")
print(report.risk_level, report.risk_score, report.indicators)
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
├── README.md              # this file
├── SKILL.md               # Agent-side description (loaded by Claude/Codex/etc.)
├── detector.py            # Main Python module — heuristics + RPC + scoring
├── scripts/
│   └── detect.sh          # Thin bash wrapper for the CLI
└── tests/
    └── test_detector.py   # 10-test pytest suite
```

## Notes

- The detector is **read-only**. No transactions are submitted.
- Default RPC is the public Pharos endpoint. For high-volume usage, point `--rpc-url` at a paid provider.
- The score is heuristic. For a security-critical decision (e.g. whitelisting a wallet to copy), still verify the contract source on PharosScan.

## License

MIT
