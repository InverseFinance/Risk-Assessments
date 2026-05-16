# rewardVault (sd-DOLA-sUSDe-vault) — Questions for the Stake DAO Team

**Asset:** Stake DAO OnlyBoost v2 RewardVault clone (`0x0c36ad1a68cdbBBFafaD7d03bb97cbaB24174e55`)
**Compiled:** 2026-05-15 by Inverse Finance Risk Working Group
**Lens:** FiRM-collateral review — can the asset's access-control surface create friction between intended liquidation value and what a liquidator actually receives?

> **TL;DR:** The asset is structurally sound for FiRM-escrow use within the 2-day Timelock window. We have **5 collaborative asks** below — primarily about Timelock observability, an audit-history disclosure, and one operational question about emergency-mode reward recovery. **None of these are blockers**, and we expect the conversation to refine FiRM's sizing and runbooks rather than change the structural picture.

> **What this document is:** a partner risk-review summary written so the team can see exactly what we examined, what we verified, what we couldn't close from public sources, and what protective additions (if any) would help us size the integration confidently.
>
> **What it isn't:** a security audit, a bug report, or a prerequisite list. The 5 asks in §2 are conversation starters, not gates.

## Contents

1. [Residual attack-vector class](#1-residual-attack-vector-class--worst-case-governance-scenarios)
2. [Protective additions we'd value from the team (5 asks)](#2-protective-additions-wed-value-from-the-team)
3. [Bottom line](#bottom-line)
4. [How to respond](#how-to-respond)

We've completed 3 cycles of on-chain + source-level verification against this vault and the contracts reachable from it through ownership, role, and module references (RewardVault, ProtocolController, Accountant, CurveStrategy, OnlyBoostAllocator, ProtocolTimelock, CurveYCRVVoter, ConvexSidecarFactory). The asset's liquidation-path integrity is **structurally sound** within the 2-day Timelock window: implementation is non-upgradeable, Accountant fees apply to rewards only (never withdraw principal), `pause()` and `shutdown(gauge)` do not block FiRM liquidator withdrawals, and the shutdown path auto-repatriates LP from all targets to the vault.

---

## 1. Residual attack-vector class — worst-case governance scenarios

The two liquidation-critical on-chain levers that remain are both gated by `ProtocolController.owner()` (= `TimelockController` 2d with Gnosis Safe 3/5 `0xB0552b6860CE5C0202976Db056b5e3Cc4f9CC765` as proposer / executor):

1. **`setStrategy(bytes4 protocolId, address strategy)`** — substitutes the custodian of every gauge position under that protocol ID
2. **`setAllocator(bytes4 protocolId, address allocator)`** — substitutes the routing-decision contract (and via its immutable `SIDECAR_FACTORY` reference, the sidecar resolution path)

Below are the two proof-of-concept call sequences the protocol's authority chain would technically permit if governance authority were operating against partner interests. Both are 2-day Timelock-buffered and analyst-observable via `CallScheduled` events, but cannot be stopped from outside the Stake DAO governance perimeter once execution fires. We share these not as accusations — your structural design correctly puts both behind a 2-day delay — but as the framing for why our 5 asks below center on observability of that window.

### 1a. Drain attack — Strategy substitution + LOCKER co-ordination

The CurveStrategy is the actual custodian of the gauge position; it's also recognized as the `strategy()` on the LOCKER (`CurveYCRVVoter` `0x52f541764E6e90eeBc5c21Ff570De0e2D63766B6`), which is the only address authorized to call `LOCKER.withdraw → gauge.withdraw`. A drain requires substituting BOTH references within the same Timelock execution window:

**Step 1 — `t = 0`.** Safe 3/5 calls `Timelock.schedule(...)` for two coordinated proposals:
- `ProtocolController.setStrategy(CURVE, maliciousStrategy)`
- `CurveYCRVVoter.setStrategy(maliciousStrategy)`

Both share the same Timelock authority chain (`owner()` / `governance()` → Safe 1/1 `0xe5d6…8b91` → Timelock 2d). `CallScheduled` events are emitted publicly at this moment.

**Step 2 — during the 2-day window (`t = 0` to `t + 2d`).** Risk partners and end users can:
- panic-withdraw / panic-liquidate
- disable the FiRM market against this collateral
- invoke `Timelock.cancel(...)` if a partner holds `CANCELLER_ROLE`
- coordinate with Stake DAO governance to abandon the proposal

**Step 3 — `t = +2d`.** `Timelock.execute()` fires both schedules. After execution:
- `PC.strategy(CURVE) = maliciousStrategy`
- `LOCKER.strategy() = maliciousStrategy`
- `maliciousStrategy` is now authorized to call `LOCKER.withdraw → gauge.withdraw` and route LP to an attacker instead of the `receiver` argument.

**Step 4 — `t = +2d + ε`.** `maliciousStrategy.drain()` is called by the attacker.
- All LOCKER-held LP is withdrawn to the attacker
- Convex sidecar LP is optionally drained via the same `maliciousStrategy` calling `ISidecar(target).withdraw` with the attacker as receiver
- User shares in the vault still exist but back zero LP
- FiRM's escrow returns 0 LP on `liquidate()` → permanent bad debt

### 1b. Block-liquidation attack — Strategy substitution to revert

A simpler variant that doesn't drain but permanently blocks every withdraw:

**Step 1 — `t = 0`.** Safe 3/5 schedules `ProtocolController.setStrategy(CURVE, revertingStrategy)` where `revertingStrategy.withdraw()` and `revertingStrategy.shutdown()` both revert unconditionally.

**Step 2 — `t = +2d`.** `Timelock.execute()` — substitution complete. Resulting state:
- Every `vault.withdraw()` reverts at `strategy().withdraw(...)` (`RewardVault.sol:383`).
- **Escape hatch is also blocked.** `ProtocolController.shutdown(gauge_)` calls `IStrategy(...).shutdown(gauge_)` (`PC.sol:350`). If `revertingStrategy.shutdown` reverts, the entire `ProtocolController.shutdown` reverts — there's no way to flip the gauge into the safe 1:1-vault-direct exit mode.
- Recovery would require another `setStrategy` via another 2-day Timelock — which a hostile governance authority would not execute.
- FiRM liquidations on this collateral revert indefinitely. Permanent bad-debt accrual on every borrower position.

### 1c. What CANNOT be done even with full Timelock + Safe compromise

| Attack vector | Why it's structurally blocked |
|---|---|
| Apply a withdraw-time fee | Accountant fee math (`src_Accountant.sol:310-376`) multiplies feePercent against `newRewards` / `newFeeSubjectAmount` only — never against withdraw amounts or share-burn quantities. The principal LP path has no fee-application site. Would require an Accountant code upgrade, but Accountant is not behind a proxy. |
| Upgrade `_withdraw()` to redirect to attacker | Implementation `0x74d8dd40118b13b210d0a1639141ce4458cae0c0` is non-upgradeable. EIP-1967 impl + admin slots both read as `0x00…00`. Bytecode is permanently 13,348 bytes of the verified RewardVault source. |
| Faster-than-2-day rug | Every consequential setter on ProtocolController is `onlyOwner` → Timelock-gated. 2 days is the floor, enforced by `TimelockController.getMinDelay()` returning 172,800. |
| Mint new vault shares directly to attacker | No admin-mint primitive exists in the implementation. `_deposit` requires actual asset transfer; `_mint` is internal and only called from within `_deposit`. |
| Block harvest to strand user yield | Yield-side concern only — `harvest()` lives in Accountant and is orthogonal to share-burn math. Liquidation principal is unaffected. |

### 1d. The geometry of the protection

The asset is bounded by the 2-day Timelock window, not by cryptographic impossibility. From a partner's perspective the relevant safeguard is **observation + reaction infrastructure during the window**, not the absence of attack vectors.

1. **`t = 0`** → Safe 3/5 schedules `setStrategy` / `setAllocator`. `CallScheduled` is emitted publicly.
2. **`t = 0 → +2d`** (the 2-day analyst-observable window). Partner-side actions:
   - Monitor `CallScheduled` events on the Timelock
   - Force or incentivize user withdraws + liquidations
   - Pause new lending against this collateral
   - Invoke `Timelock.cancel(...)` if a partner holds `CANCELLER_ROLE`
   - Coordinate with Stake DAO governance to withdraw the proposal
3. **`t = +2d`** → `Timelock.execute()`. After this point: funds drainable or liquidations blockable depending on the proposal's payload.

### 1e. Historical cadence — these levers are exercised in normal operations

We pulled the actual on-chain history of the routing-custody setters on ProtocolController to ground the discussion. **These setters are not theoretical attack surfaces — they're routine governance touchpoints exercised regularly by Stake DAO since deploy.** That actually *strengthens* the case for the asks below: there's a real cadence to monitor, not a once-in-deployment thing.

| Setter | Total calls (deploy → 2026-05-15) | First | Last | Notes |
|---|---:|---|---|---|
| `setStrategy` | 5 | 2025-08-26 | 2026-01-14 | 1 deploy-day direct call + 4 Timelock-mediated (incl. 2 in same tx for two protocol IDs) |
| `setAllocator` | 5 | 2025-08-26 | 2026-01-27 | Often co-scheduled with `setStrategy` |
| `setFactory` | 5 | 2025-08-26 | 2026-02-23 | Most-recent activity |
| `setLocker` | 1 | 2026-01-06 | 2026-01-06 | Part of the 7-setter reorg below |
| `setGateway` | 1 | 2026-01-06 | 2026-01-06 | Same reorg |
| `setAccountant` | 1 | 2026-01-06 | 2026-01-06 | Same reorg |
| `setFeeReceiver` | 1 | 2026-01-06 | 2026-01-06 | Same reorg |
| `setRegistrar` | 13 | 2025-08-26 | 2026-02-23 | Operational — adding/removing factory addresses as new deployments land |
| `setPermissionSetter` | 1 | 2025-10-27 | 2025-10-27 | Initial Timelock-mediated grant |

**Notable patterns:**

- **2026-01-06 — major routing reorg**: a single tx (`0x048e6710…`) re-set 7 distinct routing references on ProtocolController in one Timelock execution. This is exactly the kind of event where partner co-review is most valuable: one approval changes the entire custody / routing topology at once.
- **Effective `setStrategy` cadence is roughly every 7 weeks** since governance went live (2025-10-27). The **last change was 4 months ago (2026-01-14)** — current operational tempo is quieter than the historical average.
- **Co-scheduling is common**: `setStrategy` and `setAllocator` have appeared together in 3 of 4 Timelock-mediated executions, consistent with coordinated upgrades rather than adversarial single-lever swaps.

**FiRM-side implication for our monitoring posture.** We won't alarm on every `setStrategy` (that'd fire too often to be useful). The right posture is **proposal-by-proposal review during the 2-day window**, classifying each as routine (announced upgrade, expected counterparty) vs anomalous. This is why the publication ask in §2a.1 below is the foundation — without observable announcements at scheduling time, the proposal-by-proposal review can't happen.

---

## 2. Protective additions we'd value from the team

Five concrete asks, ordered by FiRM-side priority. The first three are about hardening the 2-day observation window that is the entire defense for the residual attack class above. The fourth is bug-class insurance for the non-upgradeable code. The fifth is the one operational question that affects user yield recovery during emergencies.

### 2a. Governance observability + reaction surface

1. **Public `CallScheduled` / `CallExecuted` feeds.** Do you publish or commit to publishing every Timelock proposal to a governance forum / Discord / status page at the moment of scheduling? Is there a public RSS / webhook / Snapshot mirror partners can subscribe to? The 2-day window only protects risk partners who actually see scheduled proposals — observability is the foundation.

2. **`CANCELLER_ROLE` co-share.** Would Stake DAO consider granting `CANCELLER_ROLE` on `ProtocolTimelock` (`0xb27afc7844988948FBd6210AeF4E1362bC2d8E6a`) to a multi-sig that includes risk partners (Inverse RWG, etc.)? Cancel is one-way and non-fund-moving — it can only block a proposal, never execute one — so the trust requirement is meaningfully lower than for proposers. The role currently sits with the same Safe 3/5 that also holds `PROPOSER_ROLE`, which means a malicious proposer set cannot self-cancel its own malicious proposal. A partner co-share would harden the window.

3. **Delay-floor commitment.** Is there a written / on-chain commitment that the 2-day `getMinDelay()` will not be reduced? The setter is `updateDelay(uint256)` gated by the Timelock's `DEFAULT_ADMIN_ROLE` (currently the same Safe 3/5), so a malicious governance could schedule a delay reduction first and then execute attacks faster. A signed commitment + observability on `updateDelay` proposals would address this.

### 2b. Documentation / audit

4. **OnlyBoost v2 audit coverage.** Which firms audited, when, and what scope? It would be especially helpful to know about coverage of:
   - the ProtocolController permission grid (registrar / permissionSetter / validAllocationTarget paths)
   - the Allocator routing math (`getDepositAllocation`, `getWithdrawalAllocation`, `getRebalancedAllocation`)
   - CurveStrategy's deposit / withdraw / shutdown / `_withdrawFromAllTargets` flow
   - the Safe-module pattern that lets CurveStrategy + 5 yield-side modules (RewardReceiverMigrationModule, CurveFactory, CurveDepositor, CurveAccumulator, CurveVoter) bypass standard signature flow on the Accountant Safe 1/1

   Linkable reports would be ideal but a summary is fine. This is bug-class insurance — since the implementation is permanently non-upgradeable, any latent bug is also permanent. Knowing what's been formally reviewed helps us size that risk.

### 2c. Operational — emergency recovery

5. **Un-harvested reward strandability at shutdown.** Source confirms `shutdown(gauge)` auto-repatriates LP via `_withdrawFromAllTargets`. Open question: what happens to un-harvested CRV / extra-rewards in the gauge / sidecar at the moment of shutdown? Three possibilities — (a) final-harvest-then-shutdown, (b) automatic claim during repatriation, or (c) strandable until a separate operational step. This affects FiRM-user yield-recovery during emergency mode, not principal recovery (already verified safe), but is operationally meaningful for the user experience under shutdown.

---

## Bottom line

Within source verification + the 2-day Timelock buffer, the asset's liquidation-path integrity is structurally sound for FiRM-escrow use. The residual risk class compresses to a 2-day-observable governance action requiring full Safe 3/5 + Timelock authority. The five asks above harden the observation window (1–3), insure against the latent bug class (4), and clarify emergency-mode yield recovery (5). None of them changes the structural conclusion — they refine sizing and operational runbooks.

Appreciate the team's time on these.

---

## How to respond

There's no fixed deadline. We're happy to take answers in whatever format is easiest for the Stake DAO team — direct reply to this document, the Stake DAO governance forum, Discord, or email to the Inverse Finance RWG. If any item needs more context from our side before you respond, just ask.

— Inverse Finance Risk Working Group
