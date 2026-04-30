**FiRM Collateral Pre-Screening Assessment**

frxUSD (Frax USD) — sfrxUSD (Staked Frax USD) — Frax Finance

April 23, 2026

[**Protocol Overview**](#heading=)

[Backing Model](#backing-model)

[DeFi Integrations and Liquidity](#defi-integrations-and-liquidity)

[Audits and Bug Bounty](#audits-and-bug-bounty)

[Access Control Scanner Findings](#access-control-scanner-findings)

[**FiRM Collateral Screening Framework Application**](#flags-&-observations)

[**Flags & Observations**](#flags-&-observations)

[**Recommended Timeline**](#recommended-timeline)

[**Sources**](#sources)

| VERDICT: DOES NOT MEET MINIMUM MATURITY THRESHOLD Two prescreening-level findings each independently fail the asset at this stage. (1) Lindy. Although the frxUSD proxy was deployed in January 2025, the current sfrxUSD functional implementation is post-FIP-430 (deployed September 2, 2025\) and was further modified by the January 26, 2026 authorization-module rewrite. These two upgrades replaced core functionality and therefore reset the Lindy clock. The effective age of the current sfrxUSD implementation is approximately twelve weeks; frxUSD's current implementation is \~12 weeks old counting the Jan 26 rewrite. Both are below the 6-month hard minimum. A subsequent Apr 2, 2026 frxUSD freezer-delegation upgrade is not counted as a Lindy reset (as it’s not a supply or integrity primitive) but it contributes to a broader upgrade-frequency concern documented below. (2) Single-key / upgrade-delay failure. Upgrade \+ owner \+ minter authority on frxUSD, the full custodian stack, and the ProxyAdmin are concentrated in a 3/5 Gnosis Safe. On sfrxUSD, minter authority and pricing authority are concentrated in a 3/6 Safe that is named timelockAddress() but has no delay behaviour. Both Safes have been verified on-chain to have zero enabled modules and zero guard contracts — there is no Zodiac Delay Module, no timelock, no on-signature policy check anywhere in the stack. A 3/5 or 3/6 signature set lands an arbitrary upgrade in the same block. Under RWG's standard defensive posture a minimum delay longer than our governance process is required to protect against a malicious or unintended upgrade. |
| :---: |

# **Protocol Overview**

Frax Finance is one of the earliest DeFi protocols, having launched its original fractional-algorithmic FRAX stablecoin in December 2020\. In 2025 the protocol executed a major pivot: the legacy FRAX token was replaced by a newly-deployed frxUSD token (0xCAcd...E29, deployed January 3, 2025), and the backing model was converted from fractional-algorithmic to fully fiat-redeemable 1:1 custodian-backed. Frax Inc (a U.S. public benefit corporation) was designated by [FIP-432](https://gov.frax.finance/t/fip-432-transfer-of-compliance-collateral-management-of-frxusd-to-frax-inc/3748) as the legal entity for compliance, custodian onboarding, and pursuit of a U.S. payment stablecoin charter. The Frax DAO retains governance authority and revenue rights.

frxUSD is the issuance token — transferable, non-yielding, 1:1 backed by reserves held at governance-approved custodians (Circle/USDC, BlackRock/BUIDL, Superstate/USTB, WisdomTree/WTGXX, and an additional USDB custodian). Each custodian operates an on-chain `FrxUSDCustodian*` contract with a governance-set mint cap. sfrxUSD is the staked yielding counterpart. Until September 2025 sfrxUSD was a standard OpenZeppelin ERC-4626 vault. Under [FIP-430](https://gov.frax.finance/t/fip-430-preparation-for-frxusd-payment-stablecoin-charter-compliance/3698), executed on-chain on September 2, 2025, it was restructured into a minter-model vault-style token in which `pricePerShare` is set administratively via a per-second accrual rate updated weekly by the Frax team. ChainSecurity audited this refactor. On January 26, 2026, both frxUSD and sfrxUSD had their authorization modules replaced in a single coordinated upgrade that added ERC-3009 permit and `transferWithAuthorization` functions and removed the prior `initialize()` functions.

| Attribute | Detail |
| :---- | :---- |
| **Issuer** | Frax Finance DAO — operational delegation to Frax Inc (U.S. PBC) per [FIP-432](https://gov.frax.finance/t/fip-432-transfer-of-compliance-collateral-management-of-frxusd-to-frax-inc/3748) |
| **Assets Under Evaluation** | frxUSD (stablecoin) and sfrxUSD (yielding staked counterpart) |
| **Chain** | Ethereum mainnet \+ Fraxtal; LayerZero OFT across supported chains (currently paused — see Access Control section) |
| **frxUSD Contract** | 0xCAcd6fd266aF91b8AeD52aCCc382b4e165586E29 |
| **sfrxUSD Contract** | 0xcf62F905562626CfcDD2261162a51fd02Fc9c5b6 |
| **frxUSD admin / upgrade authority (all frxUSD \+ custodian proxies)** | Gnosis Safe 3/5 at 0xB1748C79709f4Ba2Dd82834B8c82D4a505003f27 — 0 modules / 0 guards |
| **sfrxUSD timelockAddress() (despite the name, has no delay)** | Gnosis Safe 3/6 at 0x4b45D73b83686e69d08E61105FdB7F7b51f41Bc1 — 0 modules / 0 guards |
| **frxUSD current implementation** | 0x0000000048D2c8baf31742f6765383278BAda4d5 (vanity-grinded address, \~12 weeks old — Jan 26, 2026 upgrade) |
| **sfrxUSD current implementation** | 0xAad4A1D92053a62cE7a787641d8b4E5883e96700 (\~12 weeks old — Jan 26, 2026 upgrade) |
| **Mainnet Age** | Proxies \~15 months; current functional implementations \~12 weeks (both) |
| **frxUSD on-chain totalSupply** | 118.24M |
| **sfrxUSD totalSupply** | 24.62M shares |
| **Curve aggregate exit liquidity** | \~$9M |
| **Exit-liquidity to supply ratio** | \~7% |
| **Contract Upgradability** | Yes — TransparentUpgradeable proxies throughout; no timelock, no module, no guard on any upgrade path |
| **Governance Token** | FRAX (post-rebrand from FXS); veFRAX lockup for voting |

## **Backing Model** {#backing-model}

Every frxUSD token is nominally 1:1 backed by fiat-equivalent reserves at governance-approved enshrined custodians. Redemptions return the specific custodian's reserve asset (BUIDL from BlackRock, USTB from Superstate, WTGXX from WisdomTree, and so on) rather than a uniform asset like USDC. Several of these RWAs are permissioned (USTB, BUIDL, WTGXX require KYC/whitelisting), which narrows the set of arbitrageurs who can directly round-trip custodian redemptions.

FraxNet is the cross-chain deposit/redemption contract that addresses this fragmentation: it mints frxUSD against deposited USDC through an approved custodian, and routes redemptions either through a custodian or through an on-chain RWA redeemer that converts USTB or BUIDL into USDC via Superstate's $10M USDC instant-redemption buffer. Settlement uses LayerZero OFT or Circle CCTP on the destination chain. FraxNet is the primary same-day redemption rail for non-whitelisted users.

The five custodian proxies (`FrxUSDCustodianUsdc`, `FrxUSDCustodianWithOracle`, two `FrxUSDCustodian` contracts, and `FrxUSDCustodianWithReceiver`) were queried directly for their frxUSD balances. All five return **zero frxUSD held on-contract.** This is not an integrity concern per se — it is how the custodian architecture is designed: the custodian contracts hold the *off-chain* custodial reserve asset (USDC, USTB, BUIDL, WTGXX, USDB) and use their `minter()` role on frxUSD to mint/burn corresponding frxUSD. But it reinforces that frxUSD's peg is entirely contingent on off-chain custodian balances that cannot be independently verified from the on-chain state. The Comptroller 3/5 Safe itself holds 1.92M frxUSD, likely operational float for `minter_mint` distributions.

## **DeFi Integrations and Liquidity** {#defi-integrations-and-liquidity}

* **Curve (primary exit venue):** \~$9M aggregate exit liquidity, with roughly half paired against legacy FRAX and half against sUSDS. Direct USDC/USDT/ETH pairings comparatively thin.   
* **Lending:** Fraxlend, Llamalend/Resupply, and Euler are the primary third-party lending venues. [sfrxUSD-long Llamalend market](https://www.curve.finance/lend/ethereum/markets/0x3DE37c38739dFb83b7A902842bF5393040f7BF50) TVL is $16M.  
* **Aggregator Index Inclusion:** frxUSD is a constituent in Tokemak's autoUSD and baseUSD alongside crvUSD, GHO, USR, USDe, USDS, and sUSDe.  
* **Supply Location:** Majority of circulating supply sits in the sfrxUSD vault or in LayerZero OFT lockboxes.

## **Audits and Bug Bounty** {#audits-and-bug-bounty}

Frax Finance operates one of DeFi's longest-running audit programs. ChainSecurity audited the sfrxUSD FIP-430 restructure in October 2025\. Historical audits span Trail of Bits (Fraxswap, Fraxlend; 2022\) and multiple Code4rena contests. The Frax bug bounty is notable: critical exploits pay out at the lower of 10% of exploit value or $10M (in frxUSD/FRAX), with non-critical slow-drain findings at $50,000 frxUSD fixed. Scope excludes frontend/UI/server-side.

## **Access Control Scanner Findings** {#access-control-scanner-findings}

The Trustfall Access Control Scanner v5 was run against both frxUSD and sfrxUSD. The frxUSD scan traverses 13 contracts (frxUSD, ProxyAdmin, 5 custodian contracts, LayerZero OFT adapter, EndpointV2, GnosisSafe, OneSig, and one RAW unverified contract) and surfaces 11 critical-attention and 13 high-attention observations. The sfrxUSD scan traverses 8 contracts and surfaces 4 critical-attention and 8 high-attention observations. Scan integrity is Clean on frxUSD; sfrxUSD has two WARN flags on a RAW verified-but-no-source-code contract (0x8837…5B39), assumed to be a LayerZero Delegate and discussed below. OFAC/Chainalysis sanctions screening returned clean across both stacks.

**Upgrade history — interpreted per-upgrade.** This section interprets each on-chain upgrade against what it actually does, not just the function-signature diff from the proxy history.

| Date | Token | What the upgrade actually did | Material rewrite? |
| :---: | :---: | :---: | :---: |
| Jan 3, 2025 | frxUSD | Initial proxy deployment — Ownable, Permit-capable ERC-20 with a minter registry. No freeze, no pause, no burn-from-any. | — (deployment) |
| Aug 5, 2025 | frxUSD | \+9 functions (`burn(addr,amount)`, `burnMany`, `freeze`, `freezeMany`, `thaw`, `thawMany`, `pause`, `unpause`, `isPaused`). **Interpretation: the regulated-stablecoin control surface was bolted onto frxUSD.** Pre-upgrade, frxUSD had no freeze or global pause. Post-upgrade, the owner can freeze any holder, batch-freeze, destroy balances from any address, and halt transfers globally. This is the enforcement layer Frax built in preparation for the U.S. payment-stablecoin charter work under FIP-432. | Yes — added censorship and supply-confiscation primitives. |
| Sep 2, 2025 | sfrxUSD | \+12 / −4 functions. **Added**: minter module (`addMinter`, `removeMinter`, `minter_mint`, `minter_burn_from`, `minters`, `minters_array`), pricing module (`setPricePerShareIncPerSecond`, `setAllPricingParams`, `setPricePerShareStored`, `sync`, `lastSync`). **Removed**: `previewSyncRewards`, `syncRewardsAndDistribution`, old `initialize`, `initializeRewardsCycleData`. **This is FIP-430, and it is the most consequential upgrade in the entire stack.** Share price is no longer a pure function of deposits and realized rewards — it is set administratively via a per-second rate by the 3/6 Safe. The vault no longer holds frxUSD as its underlying; `totalAssets` is computed from `totalSupply × pricePerShare`. Minters can create sfrxUSD with no corresponding frxUSD deposit. | Yes — economic semantics rewritten; Lindy reset. |
| Jan 26, 2026 | frxUSD \+ sfrxUSD (paired) | \+8 / −1 on frxUSD and \+9 / −1 on sfrxUSD — added ERC-3009 meta-transaction module (`transferWithAuthorization`, `receiveWithAuthorization`, `cancelAuthorization`, `permit`) and removed the prior `initialize()` functions. **Interpretation: authorization-module swap.** Frax adopted USDC-style gasless meta-transactions. A signed authorization can now move tokens on a user's behalf with nonce-based replay protection. The `initialize()` removal is standard post-init hygiene. The upgrade does not change supply or censorship primitives, but it does introduce a new transfer side-channel that integrators' security models must account for. | Yes — new transfer authorization path introduced; Lindy reset. |
| Apr 2, 2026 | frxUSD | \+3 functions (`addFreezer`, `isFreezer`, `removeFreezer`). **Interpretation: a freezer role was carved out of the owner and assigned to a single EOA (0x8D1c…c49F, verified on-chain as `isFreezer=true`).** Prior to this upgrade, only the 3/5 Safe could freeze accounts. Post-upgrade, a delegated operations EOA can freeze without multisig signatures. The freezer role has yet to be called. **This upgrade is NOT counted as a Lindy reset** — adding a censorship-operations delegation role does not alter collateral-integrity primitives (supply, backing, or transfer semantics of the collateral itself). It is, however, a single-EOA authority with production-capable freeze power on a $118M mcap token, which is a live access-control observation. | No — operations-role delegation, not an integrity rewrite. |

**Frequency of upgrades is an independent concern.** Five implementation upgrades across the frxUSD/sfrxUSD pair in fifteen months (three of them material integrity rewrites) against a stack with zero upgrade delay is a structural mismatch. For FiRM, the defensive minimum required to respond to a bad or malicious upgrade is approximately 4.5–5 days to lower CF. A timelock of greater length on all proxy upgrade paths would be the standard ask; the current state is instant execution.

**Gnosis Safe modules and guards.** Both key Safes were queried directly via `getModulesPaginated(0x0000…0001, 100)` and the Safe guard storage slot (`keccak256("guard_manager.guard.address")-1`):

| Safe | Address | Modules enabled | Guard contract | Effective delay |
| :---: | :---: | :---: | :---: | :---: |
| frxUSD Comptroller 3/5 | 0xB1748C79709f4Ba2Dd82834B8c82D4a505003f27 | 0 (sentinel returned) | 0x0000…0000 | None |
| sfrxUSD timelockAddress 3/6 | 0x4b45D73b83686e69d08E61105FdB7F7b51f41Bc1 | 0 (sentinel returned) | 0x0000…0000 | None |

This rules out the possibility that a Zodiac Delay Module or similar mechanism might be providing off-proxy delay behaviour that Trustfall's per-contract scan could miss. Therefore there likely is no timelock anywhere in the governance chain.

The sfrxUSD `DEPRECATED__timelockAddress()` points to the 3/5 Safe; the current `timelockAddress()` points to the 3/6 Safe. This reflects a migration at some point in the contract's history from the 3/5 Safe holding pricing authority to the 3/6 Safe holding it. Four of the six 3/6 signers overlap with the 3/5 Safe; the 3/6 adds one new signer (0x381e...4b40, onboarded Feb 12, 2026).

**Minter Supply Chain — active set, corrected.** The scanner resolves 7 active minters on frxUSD: `FrxUSDCustodianUsdc` (USDC-backed, deployed Apr 2025, 5 proxy events), `FrxUSDCustodianWithOracle` (USTB-backed, deployed Jan 2025), `FrxUSDCustodianWithReceiver` (WTGXX-backed, deployed Jul 2025, 5 proxy events including a Nov 2025 `redeemForUsdcAsync` addition), `FrxUSDCustodian 0xE827` (BUIDL-backed, deployed Jan 2025), `FrxUSDCustodian 0xFE2E` (USDB-backed, deployed Jul 2025), the LayerZero `FraxOFTMintableAdapterUpgradeable` bridge (deployed Feb 2025), and the Gnosis Safe 3/5 itself. One minter (0x3c2f…2E91) was explicitly removed May 7, 2025\. All five custodian minters are themselves TransparentUpgradeable proxies — compromise of any single proxy's upgrade controller grants uncapped frxUSD mint authority, but all of them converge on the same 3/5 Safe, so the practical blast radius is the 3/5 Safe itself. The `minter_mint` function has been called 19 times as of April 15, 2026, with weekly \~33K frxUSD mints to the Comptroller Safe (apparent yield distribution for sfrxUSD).

On sfrxUSD, the scanner resolves 2 minters: the 3/6 Safe (which is also the `timelockAddress`) and a separate sfrxUSD-specific LayerZero bridge adapter (0x7311…E126). The 3/6 Safe has been the primary caller of `minter_mint` on sfrxUSD (3 calls through Oct 2025, minting \~5.3M sfrxUSD to itself without a corresponding frxUSD deposit — 1.75M of those shares are still held by the Safe as of today). Note that `minter_mint` on sfrxUSD creates shares without a frxUSD deposit, breaking the standard ERC-4626 share-price invariant. Any FiRM oracle for sfrxUSD must treat the reported exchange rate as an admin-curated value with rate-of-change caps and a tight staleness threshold.

**LayerZero Delegate**. Per Trustfall v5, 0x8837…5B39 is a LayerZero delegate on the shared EndpointV2 (0x1a44…728c) — one of 5,490 delegates registered against that endpoint, not a direct minter. Its `owner()` is a single EOA (0xe387…C5a5) and the contract has no verified source code on Etherscan (NO\_SOURCE / NO\_ABI WARN flags), which is a material observation in its own right — an unverified contract holding OApp-configuration authority on the shared LayerZero endpoint should be understood as part of the cross-chain supply plumbing, but it is not a root-token mint vector. 

**OFT is currently paused.** Per Trustfall v5 (scanned April 20, 2026), Frax cleared the peer set on the EndpointV2 on April 19, 2026 — 0 peers currently configured, which structurally disables inbound LayerZero message reception. The dormant-but-inflatable supply surface from the LZ sub-stack is inert in the current snapshot. Un-pause is a single `setPeer` transaction from the 3/5 Safe.

**Silent Setters.** The scanner flags `freezeMany`, `minter_burn_from`, and `burnMany` on frxUSD as SILENT — no event is emitted when these functions are called. Currently all dormant per Trustfall, but if used they would be invisible to event-based monitoring; history would only be recoverable via transaction scraping.

**FiRM Collateral Screening Framework Application**

The following table applies each criterion from the FiRM Collateral Screening Framework to frxUSD and sfrxUSD. Two criteria are treated as prescreening-level blockers in this assessment per RWG's standard for top-tier risk gating: Protocol Maturity (Lindy) and Single-Key / Upgrade-Delay. Both fail. Remaining items are noted for context but are not the basis for the prescreening conclusion.

| Criterion | Status | Assessment |
| :---: | :---: | :---: |
| **Protocol Maturity (Lindy)** | **FAIL** | Proxies deployed January 2025 (calendar age \~15 months), but material architectural rewrites on Sep 2, 2025 (sfrxUSD FIP-430 minter model) and Jan 26, 2026 (both contracts, ERC-3009 authorization module added) reset the effective Lindy clock. Functional implementation is \~12 weeks old on both tokens, below the 6-month hard minimum. Parent Frax organizational Lindy does not substitute for contract-level Lindy under the Framework. |
| **Single-Key / Upgrade-Delay** | **FAIL** | Two Safes concentrate admin \+ upgrade \+ mint/burn-from-any \+ pause \+ freeze authority across the entire frxUSD and sfrxUSD stack. Both Safes verified on-chain (Apr 23, 2026\) to have zero modules enabled and zero guard contracts set — confirming no timelock, no Zodiac Delay Module, and no policy check exists in the governance chain.  |
| **On-chain Track Record** | **PARTIAL** | The observed track record covers multiple different contract implementations — the post-Jan 26 code has \~12 weeks of live track record under its current authorization and transfer semantics. |
| **Security Audits** | **PASS** | ChainSecurity audited FIP-430 sfrxUSD restructure (Oct 2025). Historical Trail of Bits and Code4rena coverage of Frax infrastructure. |
| **Bug Bounty Program** | **PASS** | Critical payout: lower of 10% of exploit value or $10M in frxUSD/FRAX. Slow-drain: $50K fixed. Rewards within 5 days. |
| **Contract Upgradability** | **FLAG** | All relevant contracts are TransparentUpgradeable proxies. Frequency is the concern — 5 implementation upgrades across the frxUSD/sfrxUSD pair in 15 months, of which 3 were material integrity rewrites. |
| **Oracle Availability** | **PARTIAL** | Dedicated [Chainlink frxUSD/USD feed](https://data.chain.link/feeds/ethereum/mainnet/frxusd-usd) confirmed. Post-FIP-430 sfrxUSD requires a custom oracle: standard ERC-4626 feeds may no longer be safe since `pricePerShare` is administratively curated weekly AND the vault does not actually hold the declared underlying asset. |
| **Governance & Decentralization** | **PARTIAL** | Active FIP-driven DAO with recent proposals (FIP-418, 420, 421, 425, 426, 427, 430, 432, 434, 441). FIP-432 delegates compliance, collateral management, and IP to Frax Inc. DAO retains revocation authority but day-to-day issuer functions are centralized; admin Safe can alter protocol parameters without an FIP. |
| **On-chain Liquidity** | **PARTIAL** | \~$9M aggregate Curve exit liquidity. Thin USDC/USDT/ETH depth. Under FiRM solvency logic, any parameterization would be sized to exit liquidity, not market cap. |
| **Centralization Risk** | **FLAG** | Three overlapping concentration vectors: 100% RWA/custodian backing with permissioned redemption paths; Frax Inc (U.S. PBC) operational control of compliance and reserves; admin/upgrade authority in a 3/5 EOA Safe with no timelock. Reserve diversification across six custodians offsets single-issuer risk but not contract-level concentration. |
| **Access Control (Admin/Upgrade)** | **FAIL** | Trustfall v5 deep scan: 11 critical \+ 13 high observations on frxUSD's 13-contract stack; 4 critical \+ 8 high on sfrxUSD's 8-contract stack. No-timelock, no-module, no-guard confirmed via direct RPC query on both Safes. Multiple remediations required before any full assessment. |
| **Freeze / Blocklist Capability** | **FLAG** | Account-level freeze/thaw (consistent with regulated-stablecoin charters like USDC/BUIDL). Owner's `_update` bypass provides recovery lever that must be mapped carefully. A frozen frxUSD balance held as FiRM collateral or DOLA would interfere with liquidation paths. |

# **Flags & Observations** {#flags-&-observations}

The dominant finding is that frxUSD and sfrxUSD do not, in their current form, satisfy the FiRM Collateral Screening Framework's minimum maturity requirement, and independently fail the single-key / no-delay standard. Secondary findings cluster around oracle design, minter supply-chain breadth, and the operational reality of custodian redemption, but these are moot until the prescreening-level questions are resolved.

| Scope | Severity | Finding |
| :---: | :---: | :---: |
| **frxUSD \+ sfrxUSD** | **CRITICAL** | No timelock, no Zodiac module, no guard on any upgrade path. Both Safes verified on-chain (Apr 23, 2026\) to have 0 modules and 0 guards. 3/5 or 3/6 signatures land an upgrade in the same block. Gap between this and RWG's 5–6 day minimum defensive posture is the core single-key failure. |
| **sfrxUSD** | **CRITICAL** | `minter_mint` creates sfrxUSD without frxUSD deposit — breaks ERC-4626 share-price invariant. 5.3M shares have been minted this way historically; 1.75M of which is currently held by the 3/6 Safe. If sfrxUSD is used as collateral, share supply is not strictly backed by vault assets. |
| **sfrxUSD** | **CRITICAL** | No delay on pricing authority. The 3/6 Safe (misleadingly named `timelockAddress`) controls `setPricePerShareIncPerSecond`, `setAllPricingParams`, `setPricePerShareStored`, `addMinter`/`removeMinter` — all without delay. `pricePerShareIncPerSecond` has been changed 17 times; most recent change Apr 16, 2026\. |
| **frxUSD** | **HIGH** | 3/5 Safe concentrates admin \+ minter \+ upgrade authority across frxUSD, all 5 custodian proxies, ProxyAdmin, and the OFT adapter. Safe is also itself a minter (19 `minter_mint` calls through Apr 15, 2026). Four of five signers are EOAs with hot-wallet transaction patterns (1K–4.6K txs). |
| **sfrxUSD** | **HIGH** | Vault declares `asset() = frxUSD` but holds zero frxUSD on-contract (verified Apr 23, 2026). `totalAssets()` is derived from `totalSupply × pricePerShare`, not from actual deposits. Post-FIP-430 sfrxUSD is not a deposit-tracking vault. |
| **frxUSD** | **HIGH** | Minter blast radius: 7 active minters including 5 independent upgradeable custodian proxies. All upgrade controllers converge on the 3/5 Safe, so the effective blast radius reduces to the Safe itself, but the surface is still wide. |
| **frxUSD** | **HIGH** | Freezer role held by single EOA (0x8D1c…c49F, verified on-chain `isFreezer=true`). Owner can bypass pause and freeze checks via `_update` override. Freeze \+ burn \= full confiscation path. |
| **frxUSD** | **HIGH** | Silent setters: `freezeMany`, `burnMany`, `minter_burn_from` emit no events. Currently dormant. |

# **Recommended Timeline** {#recommended-timeline}

| Milestone | Implication for FiRM Assessment |
| :---: | :---: |
| **Do Not Advance to Full Assessment Yet** | Under the FiRM Collateral Screening Framework, the Lindy clock on both contracts has reset to Jan 26, 2026 (the most recent material integrity upgrade on either token). The 6-month hard minimum is not expected to clear before late July 2026\. Opening a full risk assessment now would presume a conclusion the Framework does not support and would set a precedent that successive upgrades can be treated as routine rather than as material changes to the asset being listed. The single-key / no-delay failure also needs remediation before advancing. |
| **Partner Outreach — Guardrail Conversation** | Use the waiting period productively to open a direct line with Frax on: (1) introducing a 5–6 day timelock on the upgrade path for all proxies (all tokens \+ all custodians \+ ProxyAdmin \+ OFT adapters), (2) hardware-attested Safe signer disclosures, (3) separation of duties between admin and upgrade multisigs, (4) advance notice on any further architectural upgrades. The goal is to lower the single-key severity before next assessment, not to renegotiate the Lindy requirement. |
| **Interim Diligence (Parallel)** | (1) Monitor mainnet on-chain liquidity profile and redemption volume. (2) Monitor `pricePerShare` cadence and `minter_mint` call patterns on sfrxUSD. (3) Assess whether the silent-setter gap (freezeMany, burnMany, minter\_burn\_from) can be mitigated via transaction-level monitoring. (4) Track `setPeer` events on both OFT adapters in case Frax re-enables the bridge. |
| **Trigger for Re-screening** | Re-screen when (a) the current implementations have accumulated at least 6 months of live operation without further architectural upgrades on either token (earliest possible: late July 2026), and (b) prerequisite remediations are in place (timelock on upgrades, signer attestations). Any new material integrity upgrade in the interim resets the clock again. Operational changes like the Apr 2, 2026 freezer-delegation role would not themselves reset the clock but would still require analyst review. |
| **If Re-screening Passes — Full Assessment Scope** | Oracle path design (Chainlink frxUSD/USD \+ custom sfrxUSD rate oracle with possible FeedSwitch v2 fallback; staleness thresholds; rate-of-change caps; acknowledgement that the vault does not hold the declared underlying asset); parameterization with stress cases for custodian impairment, FraxNet buffer depletion, and simultaneous pause, sized to the $9M exit-liquidity band not the $133M TVL band; liquidation path analysis assuming custodian redemption unavailable; centralization deep-dive on Frax Inc and admin Safe composition. |
| **If Listed Eventually — Monitoring Posture** | Beyond standard peg/liquidity checks: implementation-contract change alerts with function-count deltas treated as a reassessment trigger (supply/integrity primitives weighted heavier than operations-role additions); FIP pipeline monitoring; FraxNet/Superstate buffer status; sfrxUSD `pricePerShare` cadence and anomaly detection; freeze events; `setPeer` events on OFT adapters; and any regulatory action against Frax Inc or named custodians. |

# **Sources** {#sources}

[Frax Finance Documentation](https://docs.frax.finance/) — docs.frax.finance  
[Frax USD (frxUSD) Docs](https://docs.frax.finance/frxusd/frxusd) — docs.frax.finance  
[Staked Frax USD (sfrxUSD) Docs](https://docs.frax.finance/frxusd/sfrxusd) — docs.frax.finance  
[Frax Finance Audits](https://docs.frax.finance/other/audits) — docs.frax.finance  
[Frax Bug Bounty](https://docs.frax.finance/smart-contracts/bug-bounty) — docs.frax.finance  
[Trustfall Access Control Report — frxUSD (v5, Apr 20 2026\)](https://inverse-public-files-inverse-tools.vercel.app/vercel-report-directory/reports/Web/frxUSD.html) — https://inverse-public-files-inverse-tools.vercel.app  
[Trustfall Access Control Report — sfrxUSD (v5, Apr 17 2026\)](https://inverse-public-files-inverse-tools.vercel.app/vercel-report-directory/reports/Web/sfrxUSD.html) — https://inverse-public-files-inverse-tools.vercel.app  
[Chaos Labs — frxUSD Token Review (Oct 8, 2025\)](https://chaoslabs.xyz/posts/frxusd-token-review) — chaoslabs.xyz  
[FIP-430: sfrxUSD Payment Stablecoin Charter Preparation](https://gov.frax.finance/t/fip-430-preparation-for-frxusd-payment-stablecoin-charter-compliance/3698) — gov.frax.finance  
[FIP-432: Transfer of Compliance & Collateral Management to Frax Inc](https://gov.frax.finance/t/fip-432-transfer-of-compliance-collateral-management-of-frxusd-to-frax-inc) — gov.frax.finance  
[FIP-434: sfrxUSD AMO Deployments](https://gov.frax.finance/t/fip-434-sfrxusd-amo-deployments-future-strategies/3786) — gov.frax.finance  
[DeFiLlama — Frax USD](https://defillama.com/stablecoin/frax-usd) — defillama.com  
[Etherscan — frxUSD](https://etherscan.io/address/0xCAcd6fd266aF91b8AeD52aCCc382b4e165586E29) — etherscan.io  
[Etherscan — sfrxUSD](https://etherscan.io/address/0xcf62F905562626CfcDD2261162a51fd02Fc9c5b6) — etherscan.io  
[Etherscan — Frax Comptroller Safe 3/5 (no modules, no guard — verified Apr 23 2026\)](https://etherscan.io/address/0xB1748C79709f4Ba2Dd82834B8c82D4a505003f27) — etherscan.io  
[Etherscan — sfrxUSD timelockAddress Safe 3/6 (no modules, no guard — verified Apr 23 2026\)](https://etherscan.io/address/0x4b45D73b83686e69d08E61105FdB7F7b51f41Bc1) — etherscan.io  
[Trustfall \- Smart Contract Access Control Intelligence](https://inverse-public-files-inverse-tools.vercel.app/vercel-report-directory/index.html) — README  
*Internal: on-chain verification performed April 23, 2026 via Ethereum public RPC — covers Safe module enumeration (getModulesPaginated), guard storage slot read, sfrxUSD asset()/totalSupply()/totalAssets()/pricePerShare()/lastSync()/pricePerShareIncPerSecond()/timelockAddress()/DEPRECATED\_\_timelockAddress(), frxUSD owner()/isPaused()/isFreezer(0x8D1c…c49F), and custodian frxUSD balances. Raw output available on request.*  
