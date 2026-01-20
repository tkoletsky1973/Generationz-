# Generationz — Time-Locked BTCB Growth Vault (PCS ↔ BiSwap Arbitrage)
## Technical Specification (Dev + Audit Ready)

**Project:** Generationz  
**Type:** Time-Locked DeFi Growth Vault / Legacy Vault  
**Chain:** BNB Chain (BSC)  
**Trading Asset:** BTCB (Binance-Peg BTC)  
**Base Deposit Asset:** USDT (BEP-20)  
**DEX Venues:** PancakeSwap (PCS), BiSwap  
**Core Strategy:** Conservative buy/sell execution + PCS/BiSwap arbitrage when spread exceeds safety threshold  
**Author:** Todd Koletsky  
**Version:** 0.1.0  
**Date:** January 20, 2026  
**Status:** Draft / Integration-Ready  
**Purpose:** Generational wealth building vault with fixed maturity, penalty-based early exit, and a robust recovery system (non-custodial)

---

## Summary (Non-Negotiables)

Generationz is a **non-custodial, time-locked vault contract system** where users deposit USDT and the vault trades BTCB to generate profit using conservative buy/sell logic plus PCS/BiSwap arbitrage.

Key properties:

- **Deposit is owned by the user** (not forfeited).
- User selects a **maturity lock** (5–20 years) at deposit time.
- **Principal is locked** until maturity.
- **Early exit exists** but is intentionally painful:
  - Emergency withdrawal penalty begins at **20%** and decays monthly to **5%** by maturity.
- Profit events pay a **12% platform performance fee**.
- User selects compounding behavior:
  - Compound % from 0–100% (changeable any time).
- **Maturity withdrawal is the only penalty-free exit**.
- Withdrawals require a **Vault Unlock Code**, but code-loss is mitigated via an auditable recovery system:
  - Recovery wallet
  - Guardian threshold recovery
  - Timelocked recovery path (with notifications + friction)
- System includes **MEV/spread guards**, max slippage controls, and sanity checks.

---

# Table of Contents

1. **Concept & Product Intent**
   1.1 What Generationz Is  
   1.2 What Generationz Is Not  
   1.3 Core Design Goals  
   1.4 Threat Model (High-level)

2. **Actors & Roles**
   2.1 User  
   2.2 Protocol / Admin  
   2.3 Keeper / Automation  
   2.4 Guardians (Optional)  
   2.5 Recovery Wallet (Optional)

3. **Vault Model & Data Objects**
   3.1 Vault Definition  
   3.2 Vault Lifecycle  
   3.3 Vault Parameters (Immutable vs Mutable)  
   3.4 Vault State Machine

4. **Deposit Flow**
   4.1 Deposit Requirements  
   4.2 Maturity Selection (5–20 years)  
   4.3 Unlock Code Generation  
   4.4 Vault Binding & Recovery Setup

5. **Strategy Engine (BTCB Execution + PCS/BiSwap Arb)**
   5.1 Strategy Overview  
   5.2 Execution Gate: When Trading Is Allowed  
   5.3 Arbitrage Conditions & Spread Threshold  
   5.4 Trade Safety System  
   5.5 Slippage Limits & Price Sanity Checks  
   5.6 Blacklist & Venue Health Checks  
   5.7 Loss & Drawdown Handling

6. **Profit Event Processing**
   6.1 Definition of a Profit Event  
   6.2 12% Platform Performance Fee  
   6.3 User Compounding Selection  
   6.4 Accounting Model

7. **Maturity & Withdrawal System**
   7.1 Maturity Withdrawal (Penalty-Free)  
   7.2 Emergency Withdrawal (Penalty Applies)  
   7.3 Exit Confirmations & Friction  
   7.4 Withdrawal Fee Rules

8. **Penalty Schedule (20% → 5%)**
   8.1 Linear Decay Model  
   8.2 Monthly Update Rules  
   8.3 UI Penalty Preview Requirements  
   8.4 Anti-Gaming Rules

9. **Recovery System**
   9.1 Why Recovery Must Exist  
   9.2 Recovery Components (Code + Wallet + Delays)  
   9.3 Recovery Wallet Mode  
   9.4 Guardian Threshold Mode  
   9.5 Timelocked Recovery Mode  
   9.6 Security / Theft Resistance

10. **Automation / Keeper Layer**
   10.1 Keeper Jobs  
   10.2 Timing Rules  
   10.3 Fail-Safe Pauses  
   10.4 Manual Override

11. **Admin / Governance Controls**
   11.1 Allowed Admin Actions  
   11.2 Forbidden Admin Actions  
   11.3 Emergency Pause (Limited)  
   11.4 Upgradeability Rules

12. **Platform Fee Routing**
   12.1 Fee Destination Wallets  
   12.2 Dev/Audit/Ops/Marketing Allocation  
   12.3 Transparency & Reporting

13. **Security & Audit Notes**
   13.1 Reentrancy & External Calls  
   13.2 Oracle Manipulation  
   13.3 MEV / Sandwich Defense  
   13.4 Keeper Trust Model  
   13.5 Key Loss & Recovery Abuse  
   13.6 Testing Requirements

14. **Frontend UX Requirements**
   14.1 Screens  
   14.2 UX Tone & Friction  
   14.3 Email Reminders  
   14.4 Vault Health Dashboard

15. **Appendices**
   A. Vault State Machine Diagram Notes  
   B. Penalty Schedule Examples  
   C. Recovery Examples (User Stories)  
   D. Terms / Glossary

---

# 1. Concept & Product Intent

## 1.1 What Generationz Is

Generationz is a **long-duration, non-custodial BTCB growth vault** designed to help users commit to long-horizon wealth-building (5–20 years) without “panic selling” or short-term decision loops.

Users:
- Deposit USDT into a vault
- Select maturity lock length (5–20 years)
- Choose compounding preference (0–100%)
- Receive a vault unlock code
- Optionally configure a recovery method

The vault:
- Executes conservative BTCB trades and PCS/BiSwap arbitrage
- Takes a 12% fee on verified profit events
- Compounds or distributes profit according to user settings
- Allows exit at maturity (penalty-free) or early exit (penalty applies)

## 1.2 What Generationz Is Not

- Not a meme token
- Not a high-leverage trading platform
- Not a fixed-APY product
- Not a custodial service
- Not a guaranteed return investment
- Not a “withdraw anytime” vault

## 1.3 Core Design Goals

1. Enforce long-horizon behavior (legacy product)
2. Allow early exit but discourage it with real cost
3. Prevent lost-funds outcomes via robust recovery
4. Avoid “trust me bro” centralized custody
5. Create transparent, deterministic accounting
6. Provide extreme clarity in UX: penalty, maturity date, and recovery terms

## 1.4 Threat Model (High-level)

Assume:
- Users lose codes, phones, access to wallets, or emails
- MEV/sandwich attackers watch mempool
- Oracle manipulation attempts during thin liquidity
- DEX venue outages / routing failures
- Keeper failure / downtime
- Admin compromise attempts

The design must fail safe:
- Prefer not trading vs losing money
- Prefer delay + friction vs theft recovery
- Prefer pause vs wrong execution

---

# 2. Actors & Roles

## 2.1 User
- Deposits USDT
- Picks maturity lock
- Receives unlock code
- Selects compounding %
- Configures recovery

## 2.2 Protocol / Admin
Admin can:
- Pause execution (limited)
- Adjust strategy parameters (future trades only)
- Manage keeper list
Admin cannot:
- Withdraw user funds
- Change maturity for existing vaults
- Reduce user withdrawal rights
- Change penalty rules retroactively

## 2.3 Keeper / Automation
A keeper performs:
- market checks
- trade execution when safe + profitable
- settlement updates

## 2.4 Guardians (Optional)
User-selected addresses that can approve recovery in guardian mode.

## 2.5 Recovery Wallet (Optional)
User-selected backup wallet address that can recover vault under strict time delay.

---

# 3. Vault Model & Data Objects

## 3.1 Vault Definition

A vault is a per-user capital bucket with:
- `vaultId`
- `ownerWallet`
- `depositTimestamp`
- `maturityTimestamp`
- `principalUSDT`
- `currentBalanceUSDT`
- `strategyBalanceBTCB`
- `compoundPercent` (0–100, mutable)
- `unlockCodeHash` (never store plaintext)
- `recoveryConfig`
- `state`

## 3.2 Vault Lifecycle

1. Created at deposit
2. Active & trading
3. Settling profit events
4. Waiting until maturity
5. Exiting via maturity withdrawal OR emergency withdrawal OR recovery withdrawal
6. Closed

## 3.3 Vault Parameters (Immutable vs Mutable)

**Immutable:**
- maturity length / timestamp
- penalty decay schedule parameters (Pmax/Pmin)
- principal recorded at deposit
- unlock code hash
- initial recovery method commitments

**Mutable:**
- compoundPercent
- recovery guardian set rotations (if allowed by rules)
- recipient wallet for maturity withdrawal (optional, strict guard)

## 3.4 Vault State Machine

States:
- `CREATED`
- `ACTIVE`
- `PAUSED`
- `MATURITY_READY`
- `EXITING`
- `CLOSED`
- `RECOVERY_PENDING`

---

# 4. Deposit Flow

## 4.1 Deposit Requirements
- Min deposit: `100 USDT`
- Must pass blacklist checks (OFAC list integration optional)
- Must select:
  - maturity: 5–20 years
  - compound %: 0–100%
  - recovery setup (required to choose at least one mode)

## 4.2 Maturity Selection (5–20 years)
Allowed lock lengths:
- 60 months
- 72 months
- 84 months
- ...
- 240 months

UI must show:
- maturity date
- penalty schedule preview
- warning copy: “Early exit will cost you.”

## 4.3 Unlock Code Generation
- Generated client-side
- Displayed once
- Stored by user
- Contract stores only `hash(code)`

## 4.4 Vault Binding & Recovery Setup
User must choose at least one:
- Recovery wallet
- Guardian threshold
- Timelocked recovery path

---

# 5. Strategy Engine (BTCB Execution + PCS/BiSwap Arb)

## 5.1 Strategy Overview
Conservative execution:
- buy BTCB when conditions allow
- sell BTCB when price/target conditions allow
- arb PCS/BiSwap only when spread clears:
  - fees
  - slippage
  - safety buffer

## 5.2 Execution Gate: When Trading Is Allowed
Trades only permitted when:
- spread > minimum arb threshold
- liquidity > minimum required
- oracle sanity checks pass
- gas estimates within cap
- MEV protection enabled

## 5.3 Arbitrage Conditions & Spread Threshold
`arbSpreadNet = priceDelta - fees - expectedSlippage - safetyBuffer`

Only execute when:
`arbSpreadNet >= arbMinimumNetProfit`

## 5.4 Trade Safety System
Mandatory guards:
- max slippage
- max trade size % of vault
- max trades per day
- cooldown windows
- drawdown caps

## 5.5 Slippage Limits & Price Sanity Checks
- Router quote vs oracle must be within tolerance
- If mismatch exceeds tolerance -> no trade

## 5.6 Blacklist & Venue Health Checks
- disallow routing if venue flagged unhealthy
- disallow if liquidity below threshold

## 5.7 Loss & Drawdown Handling
If drawdown exceeds threshold:
- disable trading
- enter PAUSED
- await manual review / governance-defined action

---

# 6. Profit Event Processing

## 6.1 Definition of a Profit Event
A profit event occurs only when:
- realized gain in USDT terms > 0 after execution costs

## 6.2 12% Platform Performance Fee
Fee is applied to:
- **profit only**
Not applied to:
- principal

`platformFee = profit * 0.12`

## 6.3 User Compounding Selection
After fee:
- `profitNet = profit - platformFee`
- `compoundAmount = profitNet * compoundPercent`
- `availableToWithdraw = profitNet - compoundAmount`

`availableToWithdraw` is retained as withdrawable balance but cannot be penalty-free withdrawn until maturity.

## 6.4 Accounting Model
Must maintain ledger:
- principal
- realized profit
- fees charged
- compounded total
- withdrawable profit bucket

---

# 7. Maturity & Withdrawal System

## 7.1 Maturity Withdrawal (Penalty-Free)
Withdrawal allowed only at or after maturityTimestamp.

Requires:
- unlock code proof
- owner wallet signature
- optional 2FA/email confirmation at app layer

Closes vault permanently.

## 7.2 Emergency Withdrawal (Penalty Applies)
User may exit early.

Requires:
- unlock code proof
- owner wallet signature
- long friction confirmation flow

Penalty applies to total withdraw amount.

Vault closes permanently.

## 7.3 Exit Confirmations & Friction
Minimum friction requirements:
- 3-step confirmation
- “You will lose X%” displayed twice
- 30s countdown
- email notification sent immediately

## 7.4 Withdrawal Fee Rules
- maturity withdrawal: 0% penalty
- early withdrawal: penalty applies
- profit event fee: always applies at profit time (12%)

---

# 8. Penalty Schedule (20% → 5%)

## 8.1 Linear Decay Model
Given:
- `P_max = 0.20`
- `P_min = 0.05`
- `T = monthsTotal`
- `t = monthsElapsed`

Penalty percent:
`P(t) = P_min + (P_max - P_min) * (1 - t/T)`

## 8.2 Monthly Update Rules
- penalty recalculated on full months elapsed
- no retroactive change to model parameters

## 8.3 UI Penalty Preview Requirements
User must see:
- penalty today
- penalty next month
- penalty at maturity
- maturity date

## 8.4 Anti-Gaming Rules
- no “partial early exits”
- any early exit closes vault completely
- prevents penalty bypass strategies

---

# 9. Recovery System

## 9.1 Why Recovery Must Exist
Goal: prevent permanent lost funds from:
- lost unlock code
- lost owner wallet
- death

## 9.2 Recovery Components (Code + Wallet + Delays)
Recovery must be:
- non-custodial
- theft resistant
- slow + friction heavy

## 9.3 Recovery Wallet Mode
User sets `recoveryWallet`.

If user loses unlock code:
- recoveryWallet can initiate recovery
- triggers delay: 90–180 days
- sends notification to user email + guardians

After delay:
- recoveryWallet can withdraw to owner wallet or beneficiary wallet rules

## 9.4 Guardian Threshold Mode
User chooses:
- guardian list (N)
- threshold (M)

Recovery requires:
- M guardian signatures
- recovery delay (30–90 days)
- notifications

## 9.5 Timelocked Recovery Mode
If no guardian available:
- user initiates timelock recovery using proof-of-identity at app layer
- onchain enforcement uses delay + friction confirmations
- this is intentionally slow and annoying

## 9.6 Security / Theft Resistance
- recovery actions must be reversible during delay by original owner wallet
- all recovery actions broadcast to notification endpoints

---

# 10. Automation / Keeper Layer

## 10.1 Keeper Jobs
- evaluate market conditions
- run arb checks
- execute trades
- finalize profit events
- update vault ledger

## 10.2 Timing Rules
- trades only during safe windows
- enforce cooldowns
- enforce max executions/day

## 10.3 Fail-Safe Pauses
On repeated failures:
- auto pause
- require admin review

## 10.4 Manual Override
Admin may:
- pause all trading
- pause single asset routes
Cannot:
- liquidate vaults for profit
- withdraw vaults

---

# 11. Admin / Governance Controls

## 11.1 Allowed Admin Actions
- manage keeper whitelist
- update trading thresholds
- pause/resume trading

## 11.2 Forbidden Admin Actions
- change vault maturity
- change penalty curve for existing vaults
- remove withdrawal rights
- access unlock code data

## 11.3 Emergency Pause (Limited)
Pause stops trading only.
Withdrawals remain functional.

## 11.4 Upgradeability Rules
If upgradable:
- timelock
- multi-sig
- public notice period

---

# 12. Platform Fee Routing

## 12.1 Fee Destination Wallets
- dev
- audit reserve
- ops
- marketing/partnerships

## 12.2 Dev/Audit/Ops/Marketing Allocation
Allocations are configurable.
All changes:
- logged
- time-locked
- publicly visible

## 12.3 Transparency & Reporting
Publish:
- monthly fee totals
- active vault count
- total AUM
- performance summary

---

# 13. Security & Audit Notes

Key requirements:
- reentrancy guards
- safe external call patterns
- slippage protections
- oracle sanity checks
- MEV mitigation
- keeper permissioning
- immutable vault maturity rules
- strict recovery delays

---

# 14. Frontend UX Requirements

Required screens:
1. Landing / explainer
2. Deposit + maturity selector
3. Vault dashboard
4. Recovery setup
5. Withdraw at maturity
6. Break lock (early exit) flow
7. Notifications settings
8. Fee transparency page

Tone:
- serious
- legacy-focused
- no hype
- penalty warning language must be blunt

---

# 15. Appendices

## Appendix A — Vault State Machine Diagram Notes
(Implementation notes for Mermaid diagram)

## Appendix B — Penalty Schedule Examples
Example: 10-year vault (120 months)
- month 0: 20%
- month 12: 18.5%
- month 60: 12.5%
- month 119: ~5.125%
- month 120+: 0%

## Appendix C — Recovery Stories
- user loses phone
- user dies, beneficiary claims at maturity
- guardian recovery after loss

## Appendix D — Glossary
- Vault
- Maturity
- Unlock Code
- Profit Event
- Performance Fee
- Emergency Withdrawal / Break Lock
- Keeper
- Guardian Recovery
