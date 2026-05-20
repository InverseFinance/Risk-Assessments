# rewardVault — Strategy-Substitution Neutralization Options

**Audience:** Inverse Finance dev team (internal RWG note)
**Asset:** Stake DAO OnlyBoost v2 RewardVault clone (sd-DOLA-sUSDe-vault, `0x0c36ad1a68cdbBBFafaD7d03bb97cbaB24174e55`)
**Companion doc:** [rewardVault_team_questions.md](rewardVault_team_questions.md) (RWG → Stake DAO, partner-facing)
**Drafted:** 2026-05-20

## Context

The rewardVault team-questions doc surfaces two residual attack classes inside the 2-day Timelock window:

1. **Drain attack (§1a)** — `ProtocolController.setStrategy(CURVE, malicious)` + `CurveYCRVVoter.setStrategy(malicious)` co-scheduled. After execution, the malicious strategy can withdraw the gauge position to the attacker.
2. **Block-liquidation attack (§1b)** — `ProtocolController.setStrategy(CURVE, revertingStrategy)`. After execution, every `vault.withdraw()` reverts and the shutdown escape hatch also reverts. Permanent FiRM bad-debt accrual.

Both are gated by `ProtocolController.owner() = TimelockController (2d) 0xb27a…8E6a ← Safe 3/5 0xB055…C765` and analyst-observable via `CallScheduled`. The current 5 team asks focus on hardening the 2-day observation window. This doc explores the alternative class: **structurally neutralizing the substitution vector itself** so the 2-day window isn't load-bearing.

## Structural fact that reframes every option

`ProtocolController.setStrategy(bytes4 protocolId, address strategy)` and `setAllocator(bytes4 protocolId, address allocator)` are keyed by `protocolId`, **not by vault address**. Our vault uses PROTOCOL_ID = `0xc715e373` (CURVE), which is shared by every Curve vault Stake DAO operates (sd-crv-USD, sd-3pool, sd-stETH, etc.).

So "make setStrategy immutable for our vault" is technically "freeze the Strategy lookup for every Curve vault Stake DAO operates." That's the framing Stake DAO will hear, and the reason most options below are bigger asks than they sound.

Also worth knowing for triage: blocking either `PC.setStrategy` **or** `CurveYCRVVoter.setStrategy` is sufficient to neutralize the §1a drain attack (it requires substituting both). Blocking `PC.setStrategy` alone also defeats §1b. The cleanest neutralization target is the three `ProtocolController.set*` custody setters under PROTOCOL_ID=CURVE — `setStrategy`, `setAllocator`, `setFactory`.

## Options for neutralizing setStrategy / setAllocator

Ranked easiest → hardest for Stake DAO to entertain.

### 1. Inverse-side trustless wrapper (no Stake DAO ask required)

RWG deploys a thin escrow wrapper between FiRM and the RewardVault that:

- At deploy time, snapshots `ProtocolController.strategy(0xc715e373)` + `allocator(0xc715e373)` + (optionally) `factory(0xc715e373)` + `CurveYCRVVoter.strategy()`.
- On every `withdraw()` / `redeem()`, reverts if any live value disagrees with the snapshot.
- If a mismatch is detected, the wrapper falls back to a direct gauge withdraw path (or refuses to proceed, forcing FiRM to flip to a fallback collateral mode).

**Pros:** No trust ask on Stake DAO. Stake DAO can substitute Strategy freely for the rest of their Curve vaults — we just opt out of accepting routing changes for the escrowed LP. Self-contained risk surface.

**Cons:** Inverse engineering cost (audit, deploy, FiRM-side integration). Adds one more contract Inverse maintains. If Stake DAO ships a legitimate Strategy upgrade (security fix, gas optimization), our wrapper would also refuse it — we'd need to redeploy / re-anchor the snapshot, which is an operational task.

### 2. Filter-canceller wrapper held by Inverse

Stake DAO deploys a small contract (~100-200 lines of Solidity) that:

- Holds `CANCELLER_ROLE` on the ProtocolTimelock (`0xb27a…8E6a`).
- Exposes a `cancel(bytes32 id)` function callable only by an Inverse-controlled multi-sig.
- Internally validates that the queued operation's `target == ProtocolController` and `selector ∈ {setStrategy, setAllocator, setFactory, setLocker, setGateway, setAccountant}` with `protocolId == 0xc715e373` before forwarding the cancel to the Timelock.

**Pros:** Stake DAO retains all upgrade capability except for the CURVE custody setters we'd want to veto. Trust ask is narrow and machine-verifiable from the contract source. One small auditable contract. We could offer to author it.

**Cons:** Doesn't remove the attack vector — it gives us a 2-day-windowed veto. If we're offline or compromised during the window, the substitution still lands. Combines best with Option 1 as defense-in-depth.

**Scope wrinkle worth being transparent about.** Because the filter keys on `protocolId == 0xc715e373` (CURVE) and `setStrategy` / `setAllocator` / `setFactory` are not vault-addressable, the practical effect of this veto is "**Inverse can veto custody changes for all of Stake DAO's CURVE-namespace vaults**" — not just ours. There is no on-chain way to narrow further, because the setters don't accept a vault argument. Stake DAO will hear this scope clearly. See the [CANCELLER_ROLE scope](#cancellerrole-scope-clarifying-the-existing-2a2-ask) section below for the two framings (RWG-side vs. Stake-DAO-side) of how to position the ask.

### 3. Stake DAO adds `freezeProtocol(bytes4)` to ProtocolController and commits to calling it

Stake DAO codes + audits + governance-proposes a new `freezeProtocol(bytes4)` function on ProtocolController. Once called for `0xc715e373`, `setStrategy` / `setAllocator` / `setFactory` for that protocolId become permanently locked.

**Pros:** Hard cryptographic neutralization — no trust ask, no observation window dependency, no per-tx vigilance needed.

**Cons:** Locks routing for **every** Curve vault Stake DAO operates, not just ours. Forfeits Stake DAO's ability to ship Strategy upgrades, OnlyBoost-math improvements, or compatibility patches across their entire Curve vault stack. Significant ask. Realistically only entertain-able if Stake DAO themselves are signaling "OnlyBoost v2 is in maintenance mode and we don't expect further setStrategy on CURVE."

### 4. Renounce / burn ProtocolController ownership (transferOwnership → 0xdead)

Locks `setStrategy` + `setAllocator` + `setFactory` + `setRegistrar` + `setPermissionSetter` + `pause` + `shutdown` across **every** protocol ID (CURVE, BALANCER, AURA, …) permanently.

**Pros:** Maximum cryptographic neutralization.

**Cons:** Stake DAO loses all upgrade capability for OnlyBoost v2 forever, across every protocol and every vault. Polite no. Listed for completeness only.

### 5. Dedicated ProtocolController + new vault clone for FiRM-routed Curve exposure

Stake DAO deploys a second ProtocolController with its own ownership (could be burned), and our vault gets re-cloned pointing to the new PC. Gives us a namespace where they can freeze setStrategy without affecting their other Curve vaults.

**Pros:** Decouples our routing surface from Stake DAO's broader operational tempo. Stake DAO can still ship Strategy updates on their main PC.

**Cons:** Multi-month engineering on Stake DAO's side. They'd need to dual-maintain two PCs forever. Probably a no.

## CANCELLER_ROLE scope (clarifying the existing §2a.2 ask)

OZ `TimelockController.cancel(bytes32 id)` does a single `onlyRole(CANCELLER_ROLE)` check; there's no per-operation gating. The Stake DAO Timelock at `0xb27a…8E6a` is a single shared instance buffering **all** their governance (Curve + Balancer + Aura + permissionSetter / registrar / fee changes across every vault). Vanilla CANCELLER on that Timelock = veto power over Stake DAO's entire governance perimeter.

That's why §2a.2 in the team-questions doc is a big ask as currently phrased ("grant CANCELLER_ROLE to a multi-sig that includes risk partners").

Three sub-options for scoping it down:

| Approach | What it costs Stake DAO | What we get | Verdict |
|---|---|---|---|
| **Filter-canceller wrapper (Option 2 above)** | One small contract, auditable | Cancel power over CURVE custody setters only | **Recommended** — narrow scope, machine-verifiable |
| **Separate Timelock for FiRM-routed actions** | Deploy second TLC, dual-route their proposals, refactor governance flow | Native CANCELLER on the smaller Timelock only | Significant refactor, probably declined |
| **Co-signed canceller multi-sig (Inverse + Stake DAO signers)** | None | Coordination ritual, no actual scope reduction | Doesn't achieve the protective intent |

### The filter-canceller scope wrinkle (be honest in the ask)

The narrowest filter we can write keys on `protocolId == 0xc715e373` (CURVE). Because `setStrategy` / `setAllocator` / `setFactory` are written against a `mapping(bytes4 => address)` with no vault argument, **every Curve vault Stake DAO operates reads the same row**. So:

- Our filter cannot discriminate "operations affecting DOLA-sUSDe" from "operations affecting sd-crv-USD / sd-3pool / sd-stETH / etc."
- The practical effect of granting CANCELLER to our filter-contract is **Inverse can veto custody changes across Stake DAO's entire CURVE-namespace vault stack**, not just ours.

Stake DAO will hear that scope clearly. Two framings to offer when proposing:

**RWG framing (the case for):** The veto is over *shared infrastructure* (the CurveStrategy / OnlyBoostAllocator contract addresses that all Curve vaults depend on), not over Stake DAO's operational freedom. In normal operations, Stake DAO does not need to swap CurveStrategy — the historical cadence shows `setStrategy` is exercised every ~7 weeks, last touched 2026-01-14, and almost always co-scheduled with announced upgrades. A cancel veto on this shared infrastructure costs Stake DAO nothing in the steady state and costs them a coordination round-trip during legitimate upgrades. In exchange, every Curve vault gets a partner-monitored governance buffer — a feature Stake DAO could even market.

**Stake DAO framing (the case for caution):** Giving Inverse veto power over their other Curve customers' vaults is a non-trivial trust ask made on behalf of third parties who didn't agree to it. If Stake DAO ships a legitimate upgrade and Inverse holds it up (deliberately or by being offline / unresponsive within the 2-day window), every Curve-vault customer takes the operational hit, not just Inverse. The CANCELLER role is non-fund-moving so the failure mode is bounded — but the optics still matter.

How we'd suggest mitigating in the ask itself:

1. **Bounded cancel window** — the filter contract could refuse to cancel within (e.g.) the final 4 hours of the 2-day window, so Stake DAO has time to re-propose / re-execute if Inverse fails to respond.
2. **Public veto reason** — require the filter to emit a `VetoIssued(id, reason)` event with on-chain commentary, making vetoes auditable and creating reputational accountability.
3. **Time-boxed grant** — `CANCELLER_ROLE` on the filter-contract is granted for a fixed period (e.g. 12 months) and renews via a Stake DAO governance proposal. Forces a recurring reaffirmation.

None of the three are technically required for the filter to work — they're trust-signaling that lowers the cost to Stake DAO of agreeing.

## Recommendation

**Combine Option 1 (Inverse-side trustless wrapper) with the narrowed §2a.2 ask (Option 2 / filter-canceller).**

- The trustless wrapper neutralizes the attack class regardless of what Stake DAO does — no reliance on the 2-day observation window.
- The filter-canceller adds a partner-side veto as defense-in-depth and creates a structural coordination point with Stake DAO on CURVE custody changes.
- Both leave Stake DAO's ability to operate their broader vault stack untouched, which is the political precondition for Stake DAO actually agreeing.

If we go this route, §2a.2 of the team-questions doc should be rephrased from "grant CANCELLER_ROLE on ProtocolTimelock to a partner multi-sig" → "grant CANCELLER_ROLE to a filter-canceller contract scoped to ProtocolController custody setters with protocolId = 0xc715e373 (CURVE) — we can offer to author it."

## Open questions for the dev team

1. **Wrapper deploy budget.** Is RWG / FiRM willing to fund the engineering + audit of an Inverse-side trustless wrapper, or is the preference to push the protective surface onto Stake DAO?
2. **Legitimate-upgrade handling.** If Stake DAO ships a legitimate CurveStrategy upgrade post-wrapper-deploy, do we want (a) the wrapper to refuse and force FiRM to redeploy, (b) a governance-controlled re-anchor function on the wrapper, or (c) some other escape hatch?
3. **Filter-canceller authorship.** If we offer to author the filter-canceller, who internally would own that? FiRM smart-contract team, RWG, or a contracted auditor?
4. **Should we update §2a.2 now**, or wait for the broader team-questions doc revisit that follows the §4a internal discussion?

— Inverse Finance Risk Working Group
