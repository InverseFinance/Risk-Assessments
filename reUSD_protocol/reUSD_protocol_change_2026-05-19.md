# reUSD — Material On-Chain Protocol Change Observed (2026-05-08 → 2026-05-16)

**Asset:** Re Protocol Deposit Token (reUSD) — `0x5086bf358635B81D8C47C66d1C8b9E567Db70c72`
**Compiled:** 2026-05-19 by Inverse Finance Risk Working Group (RWG)
**Audience:** Inverse Finance team — informs the RWG decision to restart its reUSD debrief cycle and refresh the prescreening document
**Lens:** FiRM-collateral review — does the asset's access-control surface materially differ from the version we previously prescreened?

> **TL;DR:** Between our previous Trustfall scan (2026-05-08) and today's rescan (2026-05-19), the Re Protocol team substantially de-concentrated their deployer EOA `0x6C15B25E…7649`. The rotation landed across **2026-05-08 20:43 UTC → 2026-05-16 16:50 UTC — 10 coordinated transactions over 8 days**: 3 ownership transfers (Phase 1, 2026-05-08), 9 role revocations on 0x6C15, 1 role rotated to another EOA via a safe "grant new, then revoke old" sequence, a new operational key added on NAVConsumer, a previously-unknown contract deployment, and major AccessManager role-topology restructuring. Separately, the LayerZero OFT bridge was paused with peer cleared, dropping 3 contracts from the analyzable surface. **The prescreening document and the in-flight debrief cycle were both built around the pre-change role topology and are now stale.** Restarting both is recommended — but **RWG should first determine whether the migration is genuinely complete** (vs. mid-flight) before committing the cycle effort. The rate of change itself (10 governance txs in 8 days) is a FiRM-lens risk signal worth documenting regardless of cycle timing.

## Contents

1. [What materially changed on-chain](#1-what-materially-changed-on-chain)
2. [LayerZero OFT bridge — paused with peer cleared](#2-layerzero-oft-bridge--paused-with-peer-cleared)
3. [Profile staleness — what `profiles/reUSD.yaml` no longer reflects](#3-profile-staleness--what-profilesreusdyaml-no-longer-reflects)
4. [Recommended next step](#4-recommended-next-step)
5. [Verification completed for this memo](#5-verification-completed-for-this-memo)

---

## 1. What materially changed on-chain

The dominant CRITICAL pattern in the prescreening document was the deployer EOA `0x6C15B25E9750Dccb698C1a4023f34015bFe57649` holding 11 roles across 5 contracts — a single-key blast-radius that drove the prescreening's headline risk framing. **That key has been substantially descoped.**

The rotation was observed across an **8-day window: 2026-05-08 20:43 UTC → 2026-05-16 16:50 UTC** (10 coordinated transactions). Per-row dates below reflect the on-chain block where each role's holder set materially changed.

| Role | Contract | 2026-05-08 holder | 2026-05-19 status | Effective date (UTC) |
|---|---|---|---|---|
| OPERATOR_ROLE | InsuranceCapitalLayer | `0x6C15…7649` | ✅ **REVOKED** (no EOA holder) | 2026-05-16 01:32 |
| FEE_MANAGER_ROLE | DepositTokenRegistry | `0x6C15…7649` | ✅ **REVOKED** | 2026-05-13 20:14 |
| ORACLE_MANAGER_ROLE | DepositTokenRegistry | `0x6C15…7649` | ✅ **REVOKED** | 2026-05-13 20:14 |
| PAUSER_ROLE | DepositTokenRegistry | `0x6C15…7649` | 🔄 **ROTATED → `0x07e5…5F4F`** | grant 2026-05-13 14:32; revoke 2026-05-13 20:14 |
| ADMIN_ROLE | DepositTokenRegistry | `0x6C15…7649` | ✅ **REVOKED** | 2026-05-13 20:14 |
| PRICE_SETTER_ROLE | SharePriceCalculator | `0x6C15…7649` | ✅ **REVOKED** | 2026-05-13 20:14 |
| KYC_ADMIN_ROLE | KYCRegistry | `0x6C15…7649` | ✅ **REVOKED** | 2026-05-13 20:14 |
| EMERGENCY_UPDATER_ROLE | NAVConsumer | `0x6C15…7649` | ✅ **REVOKED** | 2026-05-13 20:14 |
| ADMIN_ROLE | NAVConsumer | `0x6C15…7649` | ✅ **REVOKED** | 2026-05-13 20:14 |
| KEEPER_ROLE | NAVConsumer | `0x6C15…7649` | ⚠️ Still held + **`0x3B01…FC5b` added as 2nd holder** | 2nd-holder grant 2026-05-16 16:50 |
| UPDATER_ROLE | NAVConsumer | `0x6C15…7649` | ⚠️ Still held + **`0x3B01…FC5b` added as 2nd holder** | 2nd-holder grant 2026-05-16 16:50 |

**Pattern observation:** PAUSER on DTR was rotated **safely** — the new holder (`0x07e5…5F4F`) was granted PAUSER at 14:32 UTC, then the old holder (`0x6C15…7649`) was revoked at 20:14 UTC. Both held PAUSER simultaneously for ~5h42m to avoid any pause-capability gap. This is a deliberate "grant new, then revoke old" sequence and signals coordinated governance, not a hurried compromise response.

**Net effect on `0x6C15B25E…7649`:**
- Pre-change: 11 roles across 5 contracts (InsuranceCapitalLayer, DepositTokenRegistry, SharePriceCalculator, KYCRegistry, NAVConsumer)
- Post-change: **2 roles on 1 contract** (NAVConsumer KEEPER + UPDATER only — both operational/keeper-style, not config-class)

**New address `0x3B018eA1105C5b2E14Df25e029a1C090cE54FC5b`** appears on NAVConsumer KEEPER + UPDATER as a second holder. Likely either a replacement-in-progress or an operational redundancy key. Custody attribution is unknown — to verify with the Re Protocol team.

**Other previously-flagged single-key surfaces (unchanged):**

| Holder | Role(s) | Status |
|---|---|---|
| `0x07e5faC51aD770e23F5399d51070647E16e75F4F` | PAUSER on ICL, CANCELLER on Timelock, **now also PAUSER on DTR** | Same identity, scope expanded (gained DTR PAUSER) |
| `0xEE16bE0374f2eFb34218affC1a8EbEe9310c47f8` | AM Operator (role 11723…) on InstantRedemption + PayoutTokenRegistry | Unchanged |
| `0x80a62B72dF1136aCBc57141FB67Aa46812fECAFc` | ADMIN_ROLE (AM root admin) on AccessManager — zero execution delay | Unchanged. Still single-key, still our most acute concern |
| `0x67dD3914A3c8FD627824153773117276a5E4f3A5` | KYC_PROVIDER_ROLE on KYCRegistry | Unchanged |

**Headline numeric deltas in scanner findings:**

| Metric | 2026-05-08 | 2026-05-19 | Δ |
|---|---:|---:|---|
| Contracts in scope | 15 | 12 | −3 |
| CRITICAL findings | 20 | 11 | **−9** |

The −9 CRITICAL drop is dominated by the deployer-key descope — the 11-role cluster on `0x6C15…7649` collapsed to a 3-role cluster on a different EOA (`0x07e5…5F4F`), and several of the previously-flagged roles now have no EOA holder at all. The −3 contracts-in-scope drop is from the OFT pause cascade (§2).

### Rotation transactions (executing tx batch identified)

The rotation comprised **10 coordinated transactions** across 2026-05-08 → 2026-05-16. Confirmed via `eth_getLogs` against `OwnershipTransferred`, `RoleRevoked(bytes32,address,address)`, and `RoleGranted(bytes32,address,address)` event signatures (standard OZ AccessControl + OZ AccessManager), filtered by relevant account topics across blocks 25,045,000–25,130,323. Full per-role verification belongs in the new debrief cycle; the executing batch is captured here as a starting point.

| # | Date (UTC) | Block | Etherscan tx | Scope |
|---:|---|---:|---|---|
| 1 | 2026-05-08 20:43:47 | 25,052,816 | [`0xc7727b56…`](https://etherscan.io/tx/0xc7727b56fbf1a31fa967fd87a852e55d713678f429fc583dff446e335e252cf9) | **OwnershipTransferred**: ReMintBurnAdapter `0x2BB4046…0e85`, prev=`0x6C15…7649` → new=Safe `0x8eec…fBaD` |
| 2 | 2026-05-08 20:47:11 | 25,052,833 | [`0x9f28cd36…`](https://etherscan.io/tx/0x9f28cd36366c7ccc52c4a1cc0e3c52f100fa76976792a57f9b58bd3b92fac93d) | **OwnershipTransferred**: ShareTokenMinterBurner `0x0DfB42aA…6B18`, prev=`0x6C15…7649` → new=Safe `0x8eec…fBaD` |
| 3 | 2026-05-08 20:53:59 | 25,052,867 | [`0xfa0db252…`](https://etherscan.io/tx/0xfa0db2529340ee0a085e45e1efb86ad2f2c2f4c5acdb46cd773758557867c85b) | **AccessManager ADMIN_ROLE granted to Safe** `0x8eec…fBaD` (role_id=0, `newMember=true`). The existing EOA admin `0x80a62b72…ECAFc` retains its grant — Safe added alongside |
| 4 | 2026-05-13 14:32:23 | 25,086,832 | [`0xbb2cc268…`](https://etherscan.io/tx/0xbb2cc2687c87a05af94141ff16e08a99571b9bebe1026146fc52cac9636d43f7) | **Grant** PAUSER_ROLE to `0x07e5…5F4F` on 4 contracts (reUSD DTR + reUSDe DTR + 2 others); 1 AccessManager event same tx |
| 5 | 2026-05-13 16:55:47 | 25,087,543 | [`0x4524575a…`](https://etherscan.io/tx/0x4524575a112d2edc91090c074a889466902f8354149a5fbf1ef66d67ff674629) | **NEW CONTRACT DEPLOYED**: `0x0590612e831c137c314cb2d3a7534993a1f1c984`, ownership 0x0 → Safe `0x8eec…fBaD`. Identity unknown — **investigate in cycle** |
| 6 | 2026-05-13 20:14:59 | 25,088,531 | [`0xd5d66fe9…`](https://etherscan.io/tx/0xd5d66fe973dd94f400a74965ac1ff77466459c8ca127eb17ce293d74f2586654) | **Bulk revocation**: 33 RoleRevoked events for `0x6C15…7649` across the reUSD stack (DepositTokenRegistry, SharePriceCalculator, NAVConsumer, KYCRegistry) + parallel revocations on reUSDe (junior tranche) equivalents |
| 7 | 2026-05-13 20:37:35 | 25,088,644 | [`0xb34ca69d…`](https://etherscan.io/tx/0xb34ca69db2cb938b8fd30e758c8497aca8fce334814e82f7819f72b36187b3a7) | **Major AccessManager restructure**: 38 logs in 1 tx, executed via Safe `0x8eec…fBaD`, proposed by caller EOA `0x7dd97c12…bcaa` (Safe signer or proposer; identity unconfirmed). Re-aliases role IDs + target/selector mappings under the new Safe-led AM governance |
| 8 | 2026-05-13 21:08:35 | 25,088,??? | [`0x0884c86b…`](https://etherscan.io/tx/0x0884c86b277b3a7ed2e8b8c5ed56c90b1b29239f2331fa70f2502f16d74a3f8e) | **Continued AM restructure**: 25 additional AccessManager events 31 minutes after tx #7. Content needs decode |
| 9 | 2026-05-16 01:32:23 | 25,104,458 | [`0x4dcba8e6…`](https://etherscan.io/tx/0x4dcba8e696a48babd34a1c96caf4a638a586924bda74bc813d4843dbdf2f2fe1) | **Revoke** OPERATOR_ROLE from `0x6C15…7649` on InsuranceCapitalLayer + parallel revocation on the reUSDe ICL equivalent (`0xe1886bE2…3082`); 14 AccessManager events in same tx |
| 10 | 2026-05-16 16:49:59 | 25,109,035 | [`0x8aa47506…`](https://etherscan.io/tx/0x8aa47506ade09ee445283762f504cf6b28a85192ac1e030ff6ea1bf3df0f6390) | **Grant** KEEPER_ROLE + UPDATER_ROLE to `0x3B01…FC5b` on NAVConsumer (reUSD) + the reUSDe-side NAVConsumer equivalent (`0x105f7f11…d717`). `0x6C15…7649` retains its grants on these roles — `0x3B01…FC5b` added as a 2nd holder, not as a replacement; 3 AM events in same tx |

**Mechanism distinction — ownership transfers vs role revocations:** Phase 1 (txs #1–#3) used `transferOwnership` and an AccessManager `grantRole(newMember=true)` — these are **one-shot ownership/membership changes** that move authority irrevocably to a new entity (here, the Safe). Phases 2-5 (txs #4–#10) used standard OZ AccessControl `grantRole` / `revokeRole` semantics — these are **reversible role-set adjustments** that can be undone with a counterparty grant or admin re-grant. The earlier framing of "the rotation" as a single class of events conflated these two distinct mechanisms — analytically, Phase 1's one-shot transfers are the harder-to-reverse anchor of the migration.

**Cross-tranche scope:** The 2026-05-13 batch removed `0x6C15…7649` from contracts in BOTH the **reUSD** stack AND a parallel set of contracts (DepositTokenRegistry-shape `0x9f9e2178…6351`, SharePriceCalculator-shape `0x1262a408…9915`, NAVConsumer-shape `0x105f7f11…d717`, KYCRegistry-shape `0xf682e0e4…99eb`, plus several ICL/InstantRedemption-shape contracts). This makes the rotation a **protocol-wide governance migration**, not a reUSD-specific operation — Re Protocol descoped the deployer key from their entire tranche infrastructure (reUSD senior + reUSDe junior). The new debrief cycle should investigate the reUSDe-side parallels and confirm whether the parallel contracts are indeed reUSDe.

**Sequencing observation:** The protocol granted authority to new holders **before** revoking it from the old holders — both at Phase 1 (Safe gained AM ADMIN_ROLE on 2026-05-08, 5 days before deployer-key revocations on 2026-05-13) and at the PAUSER rotation (new holder grant at 14:32 UTC, old holder revoke at 20:14 UTC, 5h42m overlap). Both halves prioritized **continuity of capability over expediency**, consistent with planned governance rather than an incident response. The 5-day gap between Phase 1 and Phase 2 also suggests the team allowed observability time before the bulk cleanup.

### How we detected this — scanner-enabled observation

This rotation became visible because the Trustfall access-control scanner was re-run against the same target 11 days after the prior published scan, and the two reports were diffed. The on-chain role topology change is **directly observable to anyone running the scanner on a cadence** — but only to those running it. This argues for **scheduled rescans** as a continuous-monitoring control for collateral assets: the difference between "noticed within 1 day" and "noticed within 11 days" is just rescan frequency. The cycle worked here because we happened to rescan during this session — a more deliberate cadence would close that gap without depending on session timing.

## 2. LayerZero OFT bridge — paused with peer cleared

Three contracts dropped from the analyzable surface between scans because the cross-chain bridge path was deactivated:

| Contract | Address | Why no longer in scope |
|---|---|---|
| ShareTokenMinterBurner | `0x0DfB42aA18CEed719617CD554304f6CA412a6B18` | Only reachable via the OFT mint path |
| ReMintBurnAdapter | `0x2BB4046022B9161f3F84Ad8E35cac1d5946e0e85` | **Paused 2026-04-18 by `0x6C15…7649`**; peer set cleared 2026-04-14 |
| EndpointV2 | `0x1a44076050125825900e736c501f859c50fE728c` | LayerZero v2 endpoint — only reachable via ReMintBurnAdapter |

This is structurally the same pattern as the frxUSD OFT pause we documented 2026-04-19 — when an OFT bridge contract pauses and clears its peer set, the BFS traversal can no longer reach the downstream dependencies. It is **not** a scanner regression.

**Implication for the cross-chain mint surface:** the cross-chain mint pathway (third route to MINTER_ROLE on the ShareToken, per our prior published report) is **currently inactive**. The CRITICAL framing for the OFT path in that report is moot for as long as the adapter remains paused. If the bridge is re-enabled, those 3 contracts return to scope and the OFT-related authority surface re-activates — RWG will treat the moment of un-pause as a monitoring trigger and re-scan immediately.

**Linking the OFT shutdown sequence to the rotation:** The OFT was deactivated in a 3-step sequence that connects directly to Phase 1 of the rotation — peer set cleared 2026-04-14 (operational lock), bridge paused 2026-04-18 (capability freeze), and finally **ReMintBurnAdapter ownership transferred to Safe `0x8eec…fBaD` on 2026-05-08 20:43:47 UTC (the first transaction of the broader rotation; see §1 tx #1)**. This positions the OFT shutdown as the leading edge of the Phase 1 migration, not a separate event. The OFT and the deployer-key descope were always one coordinated migration plan.

## 3. Profile staleness — what `profiles/reUSD.yaml` no longer reflects

The architecture bullets in the profile (which render as Protocol Context in the public report) still describe the pre-change role topology. **Specifically stale:**

- [public reUSD.md L54](https://inverse-public-files-inverse-tools.vercel.app/vercel-report-directory/reports/Markdown/reUSD.md) — "All four are administered (in whole or part) by EOA `0x6C15B25E9750Dccb698C1a4023f34015bFe57649`, which the scanner flags as CRITICAL — one private key holds 13 roles across 7 contracts in this stack." → **The scanner no longer flags 0x6C15 as CRITICAL — it holds 2 roles on 1 contract now.**
- [public reUSD.md L55](https://inverse-public-files-inverse-tools.vercel.app/vercel-report-directory/reports/Markdown/reUSD.md) — "NAVConsumer's `forceNAVUpdate` is a single-tx bypass of the Chainlink feed, gated only by EMERGENCY_UPDATER_ROLE held by the same EOA" → **EMERGENCY_UPDATER_ROLE on NAVConsumer is now revoked; no EOA holder. The bypass surface still exists in code but has no current key behind it. Verify in cycle whether the role is actually empty or rotated to a non-EOA we missed.**
- [public reUSD.md L56](https://inverse-public-files-inverse-tools.vercel.app/vercel-report-directory/reports/Markdown/reUSD.md) — "DepositTokenRegistry surfaces: deposit-token whitelist + per-token oracle pointer + collateral-eligibility flag + per-token deposit fee (cap 25%). All four are EOA-gated via different roles on the same key." → **No longer true — DTR ADMIN/FEE_MANAGER/ORACLE_MANAGER roles all revoked; only PAUSER rotated to 0x07e5.**
- [public reUSD.md L57](https://inverse-public-files-inverse-tools.vercel.app/vercel-report-directory/reports/Markdown/reUSD.md) — "OPERATOR_ROLE on ICL: held by both the deployer EOA (0x6C15B25E) and a Gnosis Safe 3/5. Partial migration." → **OPERATOR_ROLE on ICL no longer held by 0x6C15. The "partial migration" framing should be revisited — verify in cycle whether the Safe is now sole holder or a different EOA took over.**
- [public reUSD.md L63](https://inverse-public-files-inverse-tools.vercel.app/vercel-report-directory/reports/Markdown/reUSD.md) — "Custody-sweep model: withdrawToCustodian on ICL is the OPERATIONAL pattern…called frequently by the OPERATOR_ROLE EOA `0x6C15B25E`" → **0x6C15 no longer holds OPERATOR_ROLE on ICL. Verify in cycle who's actually calling withdrawToCustodian now.**
- [public reUSD.md L64](https://inverse-public-files-inverse-tools.vercel.app/vercel-report-directory/reports/Markdown/reUSD.md) — AccessManager admin-chain trace bullet. The trace identities (`0x80a62B72…` admin, `0xEE16bE03…` operator, `0x8aeb9453…` executor) **appear unchanged** but the surrounding context references the 0x6C15 deployer story that is now stale.

The 7 `expected_roles[*]` entries in the profile for roles previously held by 0x6C15 should be marked as `rotated_or_revoked_2026-05-13_to_2026-05-16` with the new holder (or empty-holder) noted, so future scans don't keep flagging the holder absence as a profile drift.

## 4. Recommended next step

The rotation is large (10 transactions, 8 days) and several pieces are still unresolved: an unidentified contract was deployed on 2026-05-13 (`0x0590612e…c984`), the AccessManager was substantially restructured in two large transactions (38 + 25 logs), and the caller EOA `0x7dd97c12…bcaa` that proposed the AM restructure has no public attribution. Before RWG commits to a cycle restart, **RWG will internally discuss next steps** — specifically:

- Is the migration genuinely complete, or is reUSD still mid-flight? The 3-day quiet since 2026-05-16 16:50 is suggestive but not definitive (Phase 1 ran 5 days before Phase 2 — long pauses are part of this migration's pattern).
- **Engage the reUSD team** to confirm migration status, surface any planned follow-on changes, and clarify identity of the unidentified contract + caller EOA. Cycle output is durable only if the new topology is stable; running a cycle against a moving target produces another stale prescreening.
- Decide whether to proceed now (faster turnaround, accept restart risk if more changes land) or wait for protocol-team confirmation of stability (durable cycle output, longer period of stale public prescreening).

Operational follow-through — profile refresh, prescreening re-issue, cycle restart, governance-velocity treatment — is tracked in RWG's internal todo list and will be sequenced by the discussion outcome above.

## 5. Verification completed for this memo

The on-chain claims in §1 and §2 above were verified via direct RPC queries today (2026-05-19). Remaining verification items will surface naturally as part of the new debrief cycle (§4c) and are not duplicated here.

- [x] **Executing tx batch identified** — see §1 "Rotation transactions" table (10 transactions across 2026-05-08 → 2026-05-16). All 10 tx hashes captured with Etherscan links.
- [x] **Per-role `RoleRevoked` event confirmation** — via `eth_getLogs` filtered by `account = 0x6C15…7649` across blocks 25,052,576 → 25,130,323. Returned 35 events: all 9 reUSD-stack roles claimed in §1 are direct hits, plus ~24 events on the parallel reUSDe-stack contracts.
- [x] **Phase 1 ownership transfers + AM admin grant** — via `eth_getLogs` on `OwnershipTransferred` (FROM 0x6C15) and AccessManager logs. 3 events confirmed on 2026-05-08 20:43–20:54 UTC; all three tx hashes captured in §1 table.

These verifications close the gap between "the scanner sees a topology change" and "we know what happened and can defend the new framing."

---

## How to use this document

This is an RWG memo for the Inverse Finance team. It supersedes the reUSD prescreening narrative as the working description until the prescreening doc is re-issued. The in-flight debrief cycle items can be retired; future debrief cycles should start from today's rescan as their baseline (not yet public — publication blocked on the refreshed prescreening landing).

*Generated 2026-05-19 from diff of the published Trustfall report (2026-05-08) against today's rescan, with on-chain confirmation via `eth_getLogs` against `RoleRevoked` and `RoleGranted` event signatures. Block range covered: 25,052,576 → 25,130,323 (~11 days, ~77,747 blocks).*
