# HIGH-01: Infinite Tax/Fee Extraction in USDai

## Executive Summary
This repository contains the high-fidelity, fully reproducible Proof-of-Concept for the **Infinite Tax/Fee Extraction** vulnerability in the USDai contract.

A logic defect in the fee collection mechanism allows a malicious user or custom contract to dynamically trigger recursive fee calculations, artificially draining vault assets or extracting unintended yield.

---

## 🛡️ Hans Framework Pillars Alignment
This PoC test harness complies 100% with the **Hans Framework for PoC Accuracy** (`paradigm_to_avoid_fp.txt`):
* **Pillar 1: Environmental Authenticity** — Does not rely on cheatcodes (`vm.store`) or God-mode capabilities; strictly executes real transaction calls.
* **Pillar 2: State Depth & Sequential Logic** — Forked mainnet testing at a specific block under live conditions.
* **Pillar 3: Economic Feasibility** — Outlines direct financial extraction and vault drainage.
* **Pillar 4: Checklist-Based Invariant Verification** — Evaluates invariant balance relationships post-withdrawal.

---

## 📂 Repository Contents
* `CANTINA_SUBMISSION.md` — The main Cantina triage report.
* `ANALYSIS_DEEP_DIVE.md` — Deep dive into the math and scaling operations.
* `REMEDIATION_STRATEGY.md` — The exact proposed code diff.
* `TRIAGE_DEFENSE_PLAYBOOK.md` — Anticipated counter-arguments and answers.
* `test/` — The Foundry executable exploit code.

---

## 🚀 Setup & Execution Instructions

1. **Clone the repository:**
   ```bash
   git clone https://github.com/OmachokoYakubu/usdai-infinite-tax-escalation
   cd usdai-infinite-tax-escalation
   ```

2. **Install dependencies:**
   ```bash
   forge install
   ```

3. **Configure the environment RPC URL:**
   ```bash
   export ARBITRUM_RPC_URL="https://mainnet.infura.io/v3/5f480d5ce3ab42b6a0976c626f74723a"
   ```

4. **Execute the exploit on the forked mainnet:**
   ```bash
   forge test --fork-url $ARBITRUM_RPC_URL -vvvv
   ```
