# reUSD — Material Protocol Change Detected — Debrief Cycle Restart Required

**Asset:** Re Protocol Deposit Token (reUSD) — `0x5086bf358635B81D8C47C66d1C8b9E567Db70c72`
**Compiled:** 2026-05-19 by Inverse Finance Risk Working Group
**Audience:** Internal RWG — informs decision to restart the reUSD debrief cycle and refresh the prescreening document
**Lens:** FiRM-collateral review — does the asset's access-control surface materially differ from the version we prescreened?

> **TL;DR:** Between our previous scan (2026-05-08) and today's rescan (2026-05-19), the Re Protocol team **substantially de-concentrated their deployer EOA** `0x6C15B25E…7649`. Nine roles were revoked outright, one was rotated to another EOA, and two retained roles got a second holder added as a backup key. Separately, the LayerZero OFT bridge was paused with peer cleared, dropping 3 contracts from the analyzable surface. **The prescreening document and the in-flight debrief cycle were both built around the pre-change role topology and are now stale.** Restarting both is recommended; the diff is large enough that a re-issue is more economical than patch-merging.

## Contents

1. [What materially changed on-chain](#1-what-materially-changed-on-chain)
2. [LayerZero OFT bridge — paused with peer cleared](#2-layerzero-oft-bridge--paused-with-peer-cleared)
3. [Profile staleness — what `profiles/reUSD.yaml` no longer reflects](#3-profile-staleness--what-profilesreusdyaml-no-longer-reflects)
4. [Recommended action items](#4-recommended-action-items)
5. [Verification checklist (executed during the debrief cycle)](#5-verification-checklist-executed-during-the-debrief-cycle)

---

## 1. What materially changed on-chain

The dominant CRITICAL pattern in the prescreening document was the deployer EOA `0x6C15B25E9750Dccb698C1a4023f34015bFe57649` holding 11 roles across 5 contracts — a single-key blast-radius that drove the prescreening's headline risk framing. **That key has been substantially descoped.**

| Role | Contract | 2026-05-08 holder | 2026-05-19 status |
|---|---|---|---|
| OPERATOR_ROLE | InsuranceCapitalLayer | `0x6C15…7649` | ✅ **REVOKED** (no EOA holder) |
| FEE_MANAGER_ROLE | DepositTokenRegistry | `0x6C15…7649` | ✅ **REVOKED** |
| ORACLE_MANAGER_ROLE | DepositTokenRegistry | `0x6C15…7649` | ✅ **REVOKED** |
| PAUSER_ROLE | DepositTokenRegistry | `0x6C15…7649` | 🔄 **ROTATED → `0x07e5…5F4F`** |
| ADMIN_ROLE | DepositTokenRegistry | `0x6C15…7649` | ✅ **REVOKED** |
| PRICE_SETTER_ROLE | SharePriceCalculator | `0x6C15…7649` | ✅ **REVOKED** |
| KYC_ADMIN_ROLE | KYCRegistry | `0x6C15…7649` | ✅ **REVOKED** |
| EMERGENCY_UPDATER_ROLE | NAVConsumer | `0x6C15…7649` | ✅ **REVOKED** |
| ADMIN_ROLE | NAVConsumer | `0x6C15…7649` | ✅ **REVOKED** |
| KEEPER_ROLE | NAVConsumer | `0x6C15…7649` | ⚠️ Still held + **`0x3B01…FC5b` added as 2nd holder** |
| UPDATER_ROLE | NAVConsumer | `0x6C15…7649` | ⚠️ Still held + **`0x3B01…FC5b` added as 2nd holder** |

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

## 2. LayerZero OFT bridge — paused with peer cleared

Three contracts dropped from the analyzable surface between scans because the cross-chain bridge path was deactivated:

| Contract | Address | Why no longer in scope |
|---|---|---|
| ShareTokenMinterBurner | `0x0DfB42aA18CEed719617CD554304f6CA412a6B18` | Only reachable via the OFT mint path |
| ReMintBurnAdapter | `0x2BB4046022B9161f3F84Ad8E35cac1d5946e0e85` | **Paused 2026-04-18 by `0x6C15…7649`**; peer set cleared 2026-04-14 |
| EndpointV2 | `0x1a44076050125825900e736c501f859c50fE728c` | LayerZero v2 endpoint — only reachable via ReMintBurnAdapter |

This is structurally the same pattern as the frxUSD OFT pause we documented 2026-04-19 — when an OFT bridge contract pauses and clears its peer set, the BFS traversal can no longer reach the downstream dependencies. It is **not** a scanner regression.

**FiRM-lens implication:** the cross-chain mint pathway (third route to MINTER_ROLE on the ShareToken, per the original prescreening) is **currently inactive**. The previous prescreening's CRITICAL framing for the OFT path is now moot for as long as the adapter remains paused. If/when the team re-enables the bridge, those 3 contracts will return to scope and the OFT-related risk surface re-activates — worth flagging as a monitoring trigger.

## 3. Profile staleness — what `profiles/reUSD.yaml` no longer reflects

The architecture bullets in the profile (which render as Protocol Context in the report) still describe the pre-change role topology. **Specifically stale:**

- [public reUSD.md L54](https://inverse-public-files-inverse-tools.vercel.app/vercel-report-directory/reports/Markdown/reUSD.md) — "All four are administered (in whole or part) by EOA `0x6C15B25E9750Dccb698C1a4023f34015bFe57649`, which the scanner flags as CRITICAL — one private key holds 13 roles across 7 contracts in this stack." → **The scanner no longer flags 0x6C15 as CRITICAL — it holds 2 roles on 1 contract now.**
- [public reUSD.md L55](https://inverse-public-files-inverse-tools.vercel.app/vercel-report-directory/reports/Markdown/reUSD.md) — "NAVConsumer's `forceNAVUpdate` is a single-tx bypass of the Chainlink feed, gated only by EMERGENCY_UPDATER_ROLE held by the same EOA" → **EMERGENCY_UPDATER_ROLE on NAVConsumer is now revoked; no EOA holder. The bypass surface still exists in code but has no current key behind it. Verify the role is actually empty or if it was rotated to a non-EOA we missed.**
- [public reUSD.md L56](https://inverse-public-files-inverse-tools.vercel.app/vercel-report-directory/reports/Markdown/reUSD.md) — "DepositTokenRegistry surfaces: deposit-token whitelist + per-token oracle pointer + collateral-eligibility flag + per-token deposit fee (cap 25%). All four are EOA-gated via different roles on the same key." → **No longer true — DTR ADMIN/FEE_MANAGER/ORACLE_MANAGER roles all revoked; only PAUSER rotated to 0x07e5.**
- [public reUSD.md L57](https://inverse-public-files-inverse-tools.vercel.app/vercel-report-directory/reports/Markdown/reUSD.md) — "OPERATOR_ROLE on ICL: held by both the deployer EOA (0x6C15B25E) and a Gnosis Safe 3/5. Partial migration." → **OPERATOR_ROLE on ICL no longer held by 0x6C15. The "partial migration" framing should be revisited — is the Safe now sole holder, or did a different EOA take over?**
- [public reUSD.md L63](https://inverse-public-files-inverse-tools.vercel.app/vercel-report-directory/reports/Markdown/reUSD.md) — "Custody-sweep model: withdrawToCustodian on ICL is the OPERATIONAL pattern…called frequently by the OPERATOR_ROLE EOA `0x6C15B25E`" → **0x6C15 no longer holds OPERATOR_ROLE on ICL. Check who's actually calling withdrawToCustodian now.**
- [public reUSD.md L64](https://inverse-public-files-inverse-tools.vercel.app/vercel-report-directory/reports/Markdown/reUSD.md) — AccessManager admin-chain trace bullet. The trace identities (`0x80a62B72…` admin, `0xEE16bE03…` operator, `0x8aeb9453…` executor) **appear unchanged** but the surrounding context references the 0x6C15 deployer story that is now stale.

The 7 `expected_roles[*]` entries in the profile for roles previously held by 0x6C15 should be marked as `rotated_or_revoked_2026-05-08_to_2026-05-19` with the new holder (or empty-holder) noted, so future scans don't keep flagging the holder absence as a profile drift.

## 4. Recommended action items

### 4a. Restart the reUSD debrief cycle

The in-flight debrief cycle (paused at item 12 on 2026-05-08 per memory) was constructed around the pre-change role topology. Items 1-11 verified against a now-stale state; items 13+ have not been touched. **Recommend full restart** with the new role topology as baseline rather than patch-merging old verifications.

### 4b. Refresh the prescreening document

The prescreening narrative's headline risk framing — "deployer EOA holds 11 roles across 5 contracts" — is the dominant story across the document. Post-change, that story is "deployer EOA descoped to 2 operational roles on 1 contract; primary CRITICAL surface is now the AccessManager root admin EOA `0x80a62B72…` with zero execution delay (unchanged)." That's a materially different document. **Re-issue rather than amend.**

### 4c. Update `profiles/reUSD.yaml`

- Refresh the 6 architecture bullets identified in §4 to reflect the post-change topology
- Update `expected_roles[*]` entries for the 9 revoked roles
- Add `0x3B018eA1…FC5b` as a known holder for NAVConsumer KEEPER + UPDATER (custody attribution TBD)
- Add a `protocol_change_log:` note or similar dated record of the 2026-05-08 → 2026-05-19 rotation event for future audit trail

## 5. Verification checklist (executed during the debrief cycle)

The new debrief cycle (§4a) will naturally cover these verifications as it walks today's report — they are not a separate workstream. Listed here as a preview of cycle scope so any item that exits the cycle unchecked is an explicit accountability gap.

- [ ] Run `eth_getLogs` on the 9 revoked roles' `RoleRevoked` events between blocks ~25052576 and ~25130323 to confirm the revocations and identify the executing tx batch *(/debrief Section 4b + 11)*
- [ ] `hasRole(EMERGENCY_UPDATER_ROLE, <any_addr>)` on NAVConsumer — confirm whether the role is genuinely empty or just no longer held by 0x6C15 *(/debrief standard verification habit)*
- [ ] `eth_getCode(0x3B018eA1105C5b2E14Df25e029a1C090cE54FC5b)` — confirm EOA (0 bytes) vs Contract *(/debrief standard verification habit)*
- [ ] Look up the executing tx on Etherscan: who signed the revocations? Was it the AccessManager root admin `0x80a62B72…`, the deployer key `0x6C15…7649` itself, or a Safe? *(/debrief Section 11)*
- [ ] Cross-reference against Re Protocol governance forum / Discord announcements 2026-05-08 to 2026-05-19 for stated context *(analyst-initiated during cycle)*

These verifications close the gap between "the scanner sees a topology change" and "we know what happened and can defend the new framing."

---

## How to use this document

This is an internal RWG memo. It supersedes the reUSD prescreening narrative as the working description until the prescreening doc is re-issued. The in-flight debrief cycle items can be retired; future debrief cycles should start from today's rescan as their baseline (not yet public — publication blocked on the refreshed prescreening landing).

*Generated 2026-05-19 from diff of published 2026-05-08 report against today's rescan. Block range covered: 25052576 → 25130323 (~11 days, ~77,747 blocks).*
