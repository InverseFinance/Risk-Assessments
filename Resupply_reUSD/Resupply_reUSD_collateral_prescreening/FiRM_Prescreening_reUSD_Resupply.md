# FiRM Collateral Pre-Screening Assessment

## reUSD — Resupply Protocol

**July 29, 2026**

---

- [**Protocol Overview**](#protocol-overview)
- [**DeFi Integrations and Liquidity**](#defi-integrations-and-liquidity)
- [**Audits and Bug Bounty**](#audits-and-bug-bounty)
- [**Access Control Scanner Findings (2026-07-29)**](#access-control-scanner-findings-2026-07-29)
  - [Pricing Authority](#pricing-authority)
  - [Upgrade Authority](#upgrade-authority)
  - [Mint and Supply Authority](#mint-and-supply-authority)
  - [Mint and Redemption Halt Authority](#mint-and-redemption-halt-authority)
  - [Effective Authority (nominal vs effective)](#effective-authority-nominal-vs-effective)
  - [Market Feed Independence](#market-feed-independence)
  - [Monitoring Surface](#monitoring-surface)
- [**Framework Findings and Remediation**](#framework-findings-and-remediation)
- [**Sources**](#sources)

---

> [!WARNING]
> **PROVISIONAL VERDICT: CONDITIONAL**
>
> Thirteen months past the June 2025 exploit reset, Resupply presents a credible remediation and re-audit record (six reports from ChainSecurity and Electisec, zero unresolved critical or high findings), and a clean permissioning profile with no EOA role holders. An operative FiRM-side price feed has been live since April 2025. No criterion fails the asset outright on qualification grounds. The document flags three structural concerns that together cap market sizing rather than disqualify:
>
> 1. The operative FiRM ChainlinkCurveFeed is architecturally sound and battle-tested elsewhere on FiRM, but its EMA source pool holds only 7.58M USD and sits 74/26 reUSD-heavy, so both manipulation cost and price quality degrade with a depth that is currently shrinking. Until a minimum source-pool depth threshold is defined, below which the market pauses or the feed is reviewed, the pricing pipeline cannot be relied on at any meaningful market size. This compounds the exit liquidity concern: the same thinning pool undermines both liquidation flow and price discovery.
> 2. The 3-of-5 operations Safe holds a hookless, zero-delay `setVoter` grant that collapses the 8-day governance path in a single transaction and hands over the entire Core authority, a capture our independent feed can only detect after the fact, not prevent. The surface is narrower than it first reads, since the mint and supply spine is immutable and the Safe cannot re-grant itself powers, but there is no on-chain recovery behind the Safe. Two asks to the Resupply team follow: as priority, revoke the `setVoter` grant and retire it behind the 8-day Voter path, a single `setOperatorPermissions` revoke with no new code and emergency pause preserved; as secondary, delay the GuardianUpgradeable upgrade grant, a nice-to-have since FiRM's independent feed already covers the oracle-repoint harm.
> 3. No active, well-defined bug bounty program was located, against a framework baseline that requires an active bounty proportional to TVL. Confirming host, scope, and maximum payout, or standing a program up if none exists, is a precondition to any FiRM listing.
>
> Recommended disposition: hold at pre-screening with the remediation asks below; any near-term listing should be sized against pairing depth, not headline TVL.

---

## Protocol Overview

Resupply Finance is a CDP stablecoin protocol from the Convex and Yearn ecosystem (a SubDAO credited to C2tP and collaborators). Users deposit yield-bearing lending market shares, CurveLend crvUSD vault tokens or Fraxlend frxUSD vault tokens, into one of 19 registered ResupplyPair markets (10 active) and borrow reUSD against them up to per-market debt ceilings. The borrow rate is derived from the underlying lending rate, the sfrxUSD rate, and a 2% constant, scaled by a `priceWeight` factor between 50 and 60% that rises as reUSD trades below peg. Peg defense rests on a socialized redemption mechanism (RedemptionHandler, 1% base fee, permissionless when reUSD trades below roughly 0.985) and on the Insurance Pool, a first-loss buffer of user-deposited reUSD that the LiquidationHandler burns against bad debt. Governance follows the Prisma Core and Voter pattern: a single Core ownership hub reachable either through an 8-day on-chain RSUP vote or through instant operator grants, of which 24 are live and none carries an AuthHook delay.

In June 2025 the protocol suffered a 9.5M USD loss when a freshly deployed, near-empty market was drained via an ERC-4626 donation attack: the attacker inflated the empty collateral vault share price so the solvency check divided to zero and every position appeared infinitely solvent, then borrowed 10M reUSD against one wei of collateral. The loss was socialized (6.0M reUSD burned from the Insurance Pool, roughly 2.86M in treasury and personal donations, 1.13M carried and repaid by August 2025).

| Attribute | Detail |
|:---|:---|
| **Issuer** | Resupply Finance (Convex/Yearn ecosystem SubDAO) |
| **Asset under evaluation** | reUSD, `0x57aB1E0003F623289CD798B1824Be09a793e4Bec`. Non-proxy ERC-20 inheriting LayerZero OFT; zero cross-chain peers configured (bridge scaffolded, dormant). |
| **Scan surface** | 26 contracts, 138 role slots, 161 privileged functions, 0 EOA role holders (Trustfall, block 25634777). |
| **Admin / upgrade authority** | Prisma-style Core hub. Nominal path: 8-day Voter (7-day vote, 30% quorum, 1-day execution delay). Effective fast path: 3-of-5 ops Safe holding an instant `setVoter` grant, the UpgradeOperator manager role over two UUPS proxies, and the guardian role. |
| **Mainnet age (Lindy)** | Launched March 13, 2025 (16.5 months). Maturity clock reset June 26, 2025 (exploit); 13.1 months elapsed since reset. |
| **Supply / backing** | 32,059,835 reUSD at pin. CDP-backed by CurveLend and Fraxlend vault shares, fully on-chain. Aggregate borrow ceiling 634.9M reUSD across 19 markets, roughly 5% utilized. |
| **Aggregate exit liquidity** | Roughly 11.1M USD across four Curve pools; non-reUSD pairing depth roughly 3.13M USD. All pools reUSD-heavy at roughly 72/28. |
| **Oracle / NAV pipeline (FiRM side)** | Inverse ChainlinkCurveFeed (reUSD / USD, 18 decimals), deployed April 2025: Curve reUSD/scrvUSD `price_oracle(0)` EMA combined with the crvUSD/USD Chainlink wrapper. Reading 0.99230 at pin. Staleness threshold 86,460s inherited from the crvUSD/USD feed. |
| **Redemption** | RedemptionHandler, 1% base fee, 0.5/0.5 split. Permissionless below roughly 0.985; above that restricted to the RedemptionOperator arb proxy. Guard settings instantly togglable by the ops Safe via a live Guardian wildcard grant. |
| **Governance token** | RSUP, staked via GovStaker for Voter weight. Vest manager concentration is 56.90% (43.072m is 56.90% of 75.693m). |
| **Permissioning** | User paths (borrow, repay, redeem below peg, liquidate, Insurance Pool deposit/withdraw) fully permissionless. No whitelist, no blocklist, no freeze on the token. |
| **Sanctions screening** | 56 addresses screened against OFAC SDN and Chainalysis; 0 flagged (Trustfall, cache 2026-07-28). |

## DeFi Integrations and Liquidity

All meaningful reUSD liquidity sits in four Curve StableswapNG pools on Ethereum mainnet. GeckoTerminal additionally surfaces eight Uniswap v4 micro-pools, each under 2.3k USD with zero volume; they are excluded from the table.

| Venue / Pool | TVL | Pairing Depth | 24h Volume | Notes |
|:---|:---|:---|:---|:---|
| Curve reUSD/scrvUSD | $7.58M | $2.16M (scrvUSD) | $72.3K | Primary venue; the EMA source pool for the FiRM reUSD/USD feed. 74/26 reUSD-heavy by token count (5.47M reUSD / 1.94M scrvUSD). |
| Curve reUSD/sfrxUSD | $3.01M | $0.83M (sfrxUSD) | $29.8K | Secondary venue. 76/24 reUSD-heavy (2.19M reUSD / 0.69M sfrxUSD). |
| Curve reUSD/fxUSD | $0.52M | $0.14M (fxUSD) | $2.8K | fxUSD is f(x) Protocol. StableswapNG, A=200, deployed April 2025. 73/27 reUSD-heavy. |
| Curve reUSD/sDOLA | $22.4K | $7.6K (sDOLA) | $0.4K | Dust. Inverse-adjacent pool from the 2025 LP listing work. |
| **Total (4 Curve venues)** | **$11.14M** | **$3.13M** | **$105.3K** | All pools roughly 72/28 reUSD-heavy in aggregate. |

**Composition and shape.** Today every pool is imbalanced toward reUSD at roughly 3:1. Pairing depth (the gross non-reUSD side) is the hard ceiling on aggregate exit before major slippage: roughly 3.13M USD protocol-wide, of which 2.16M sits in a single pool. CoinGecko 2% depth on the primary pool was roughly 136K USD per side when sampled July 21. Against the April 2025 baseline in our own LP assessment (68.8M USD Curve TVL, 15.5M USD paired depth on the primary pool alone), cumulative TVL has declined by roughly 84%.

**Persistence.** Between the July 21 and July 29 reads the primary pool imbalance worsened from 67/33 to 74/26 while the scrvUSD side shrank from 2.08M to 1.94M tokens. Liquidity is thinning.

**Pricing reads by integrators.** The distribution of supply is itself an integration map: 65.9% of all reUSD (21.13M) sits in the sreUSD savings vault, an ERC-4626 auto-compounder over protocol revenue with no supply capability of its own; 7.8% in the Insurance Pool; 25.2% across the four Curve pools; roughly 1.1% free float. sreUSD is itself used as collateral in a Resupply market (crvUSD/sreUSD pair, 7.6M borrowed, 30.6% utilized), a recursive structure worth noting. Tangent is the sole external lending market with reUSD listing (in LP form paired with scrvUSD and fxUSD both markets with 84% CF) with ~$480k in TVL and $350k borrowed against. Cross-chain: the LayerZero OFT wiring exists but has zero peers configured on every tested chain; a first `setPeer` event would be a supply-surface change and is on the monitoring list.

## Audits and Bug Bounty

Pre-launch, the protocol was audited by yAudit/Electisec (January 2025; 3 critical and 3 high, all fixed) and ChainSecurity (February 2025; 0 critical or high, 3 accepted low-severity risks including potential collateral insolvency in the pair). The June 2025 exploit vector was in scope for both and flagged by neither. The post-exploit re-audit record is the more relevant evidence and it is substantial: six engagements across the two incumbent firms, all with zero unresolved critical or high findings.

| Engagement | Date | Findings | Scope / Note |
|:---|:---|:---|:---|
| Electisec Update Report (exploit fixes) | Jul 14-15, 2025 | 0C / 0H / 0M / 0L / 4 info | Exchange-rate fixes in ResupplyPairCore, updated PairDeployer, new BorrowLimitController (7-day up-only ceiling ramps). |
| ChainSecurity Resupply v2 | Aug 12, 2025 | 0C / 0H / 6M (all corrected) / 20L | Share-price inflation mitigations and BorrowLimitController. Repo PDF is stamped PRIVATE / DRAFT [OPEN: request the final public version]. |
| ChainSecurity sreUSD | Aug 19, 2025 | 0C / 0H / 2M / 5L | sreUSD vault, PriceWatcher, InterestRateCalculatorV2, fee distribution. Public. |
| ChainSecurity CurveLend Operators | Oct 15, 2025 | 0C / 0H / 1M (corrected) | CurveLend minter factory and operator. Public. |
| Electisec CurveLend Operator | Nov 24, 2025 | Low / informational only | 2 contracts, 120 LOC. |
| Electisec sreUSD | Nov 25, 2025 | 0C / 0H / 1M / 3L / 1 info | 7 contracts, 750 LOC. Medium: PriceWatcher weight can exceed the 1e6 scale. |

No active, well-defined bug bounty program was located this cycle. Under the framework, an active bounty proportional to TVL is a baseline control.

## Access Control Scanner Findings (2026-07-29)

![reUSD: what can be upgraded, and what it can reach. The three upgradeable proxies (GuardianUpgradeable, RedemptionOperator, TreasuryManager) are operational role-holders under the 3-of-5 operations Safe; the supply spine (Oracles, Core, ResupplyRegistry, reUSD token) is immutable and non-proxy. Only GuardianUpgradeable bridges to the token's economics, via an oracle repoint.](assets/fig1_upgrade_scope.png)

### Pricing Authority

Resupply's own peg oracle (ReusdOracle) is fully immutable but is floored at 1 minus the redemption fee, meaning it can never print below roughly 0.99; it is a redemption-arbitrage price, not a market price, and is informational only from FiRM's perspective since no FiRM contract would consume it. The protocol-internal repoint lever (`REUSD_ORACLE` registry key) is Core-only on an 8-day path and guarded against the Guardian fast route, but two instant single-actor bypasses exist, both held by the 3-of-5 ops Safe: the `setVoter` grant, and a hostile GuardianUpgradeable upgrade that drops the guard check inside the implementation and repoints the key. These bypasses threaten Resupply's internal solvency machinery, not FiRM's feed, and are graded under Effective Authority.

### Upgrade Authority

Three UUPS proxies exist; everything else in the stack, including the entire mint and supply spine (the reUSD token, the ResupplyRegistry as sole mint operator, Core, and both oracles), is non-proxy and cannot be upgraded. A hostile upgrade therefore cannot rewrite mint logic and cannot grant itself new powers: new grants require `setOperatorPermissions`, which is Voter-only, and the ops Safe holds neither the target-specific nor the wildcard form of that grant (verified live at block 25646735). The blast radius of any upgrade is capped at the grants the proxy already holds, exercised arbitrarily. Of the three proxies, the Safe can fast-upgrade two through the UpgradeOperator (manager role, no AuthHook, no delay): GuardianUpgradeable, graded CRITICAL because a malicious implementation drops the `guardedRegistryKeys` check and repoints `REUSD_ORACLE`, and RedemptionOperator, graded HIGH but bounded and anti-dilutive, since redemption burns reUSD and the proxy holds no mint right. TreasuryManager carries no fast grant, is upgradeable only via the 8-day Voter path, and already satisfies the delay standard on its own. Through the FiRM lens, the Guardian vector is mitigated on our side: FiRM prices reUSD from our own market-based feed, not `REUSD_ORACLE`, so an oracle repoint inside Resupply cannot misprice collateral on our ledger. That concentrates the residual concern on the `setVoter` grant covered under Effective Authority, which an independent feed can only detect after the fact, not prevent. The Guardian proxy was upgraded once on 2026-03-16 (+1 function, pause-related); intent unverified but consistent with routine maintenance.

### Mint and Supply Authority

reUSD mint routes through a single chokepoint: the token's only operator is the ResupplyRegistry, whose `regPair` gate admits only registered pairs, each bounded by its own `borrowLimit` plus `maxLTV` and oracle solvency. Registering a new pair (`Registry.addPair`) is Core-only on the 8-day path; ceiling increases ramp up-only over at least 7 days via BorrowLimitController. Two caveats: the `withdrawFees` path mints accrued interest as new reUSD outside the `borrowLimit` accounting, and the aggregate pre-approved ceiling is 634.9M reUSD against 32.1M live supply, a 20x overhang that makes the nominal cap economically meaningless as a bound. The RSUP emissions stack (GovToken, EmissionsController, GovStaker) has no reUSD mint capability; scanner edges suggesting otherwise are transitive getter-pointer over-scan artifacts, per the reviewed profile.

### Mint and Redemption Halt Authority

The Guardian role (held by the ops Safe) can pause any or all pairs instantly, which zeroes `borrowLimit` and freezes new minting but never blocks repay, redeem, or liquidate; collateral cannot be trapped by a pause. The same Safe can instantly disable the permissionless below-peg redemption backstop via a live wildcard grant on `updateGuardSettings`, and Core can raise the base redemption fee to 100% on the 8-day path, which would make redemption economically worthless. The Insurance Pool's exit can also be frozen by the Guardian. `pausePair` has fired 8 times historically, consistent with the market wind-downs visible in the registry.

### Effective Authority (nominal vs effective)

Nominal authority is the 8-day Voter with a real and clean history: 24 proposals, 23 executed, 0 cancelled. Effective authority is the 3-of-5 ops Safe, which holds a live, hookless, instant `setVoter` grant on Core (set at block 23225032). One multisig transaction reassigns the supreme arbitrary-execute authority, after which a second `Core.execute` call reaches any lever in the stack, including `reUSD.setOperator` and direct mint. The same Safe is simultaneously the guardian (pause, redemption guard, proposal veto), the UpgradeOperator manager (two UUPS proxies), the TreasuryManager manager, and the sole approved deployer slot on the PairDeployer. Veto power and capture power therefore share one custodyl. There is no on-chain recovery once `setVoter` fires; the mitigants are the 3-of-5 quorum and the second-transaction requirement for mint specifically. This is the single most consequential finding in the profile.

### Market Feed Independence

FiRM would not consume any Resupply-controlled oracle. The operative pipeline is our own ChainlinkCurveFeed, configuration verified on-chain this cycle: `curvePool` is the reUSD/scrvUSD pool, `assetToUsd` is the crvUSD/USD Chainlink wrapper, `assetOrTargetK` 0, `targetIndex` 0. The feed contract exposes no setters; its parameters are constructor-fixed. The FiRM feed's independence from Resupply governance is structural: neither the Curve pool EMA nor the Chainlink crvUSD/USD wrapper is touchable by any Resupply lever. The dependency runs the other way, through market structure: the EMA source pool holds 7.58M USD and is 74/26 imbalanced, so the cost of bending the EMA scales with a depth that is currently shrinking. The feed's floor behavior under a true depeg would track the market, unlike Resupply's floored internal oracle. Per-pair collateral oracles inside Resupply (BasicVaultOracle, a bare `convertToAssets` read with only an upper bound) remain the protocol's soft spot for newly added markets; existing seasoned markets are insulated by their populated vaults.

### Monitoring Surface

Should this proceed toward listing, the weekly Risk Observer Checklist additions are: Core `OperatorSet` events targeting selector `0x4bc2a657` (`setVoter`) and `VoterSet` events on Core (the fast-path tripwire); upgrade events on the Guardian and RedemptionOperator proxies; Registry `addPair` and `setRegistryAddress` events; first `setPeer` on the reUSD or sreUSD OFT (cross-chain surface activation); Insurance Pool balance and share of supply; primary pool balance ratio and pairing depth; `updateGuardSettings` events on the RedemptionHandler; and BorrowLimitController ramp arming. The reUSD/USD feed heartbeat inherits the crvUSD/USD staleness window (86,460s) and should be added to the feed monitor with the standard variance alert.

## Framework Findings and Remediation

Statuses: 🟢 **PASS**, 🟠 **FLAG** (concern that shapes parameters or sizing), 🟡 **EARLY** (insufficient history), 🔴 **FAIL** (disqualifying until remediated). Remediation entries are concrete asks to the Resupply team or internal follow-ups; rows with no ask carry a dash.

| Criterion | Status | Assessment | Remediation |
|:---|:---:|:---|:---|
| **1a. Oracle Integrity** | 🔴 **FAIL** | Operative pipeline is the FiRM ChainlinkCurveFeed (deployed April 2025, reading 0.99230, config verified on-chain at pin). Architecture is sound and battle-tested elsewhere on FiRM, but the EMA source pool is 7.58M USD and 74/26 imbalanced, so manipulation cost and price quality degrade with a depth that is currently shrinking. Staleness inherited from crvUSD/USD (86,460s). | Internal: define a minimum source-pool depth threshold below which the market pauses or the feed is reviewed; add pool ratio to the weekly checklist. |
| **1b. Pricing Authority** | 🟢 **PASS** | The FiRM feed exposes no setters and no Resupply lever reaches it. Resupply's own ReusdOracle is immutable but floored at roughly 0.99 and is informational only from FiRM's perspective; no FiRM contract would consume it. | - |
| **2. Upgradability / Effective Authority** | 🟠 **FLAG** | Nominal 8-day Voter governance collapses to a single 3-of-5 Safe transaction via the hookless `setVoter` grant, the apex lever: a hostile Voter reaches the entire Core authority (mint operators, ceilings, every oracle, upgrade rights) with no delay. The upgrade surface itself is narrower than it first reads: the mint and supply spine is immutable, the Safe cannot re-grant itself powers (no `setOperatorPermissions` grant, verified at block 25646735), and of the two fast-upgradeable proxies only GuardianUpgradeable bridges to reUSD economics, via oracle repoint, a vector FiRM's independent feed already mitigates on our side. Instant, un-timelocked, no on-chain recovery. Quorum-tempered and historically unused (`setVoter` fired zero times) | Ask team, priority: revoke the `setVoter` grant, retiring it behind the 8-day Voter path. A single `setOperatorPermissions` revoke, no new code; emergency pause is a separate grant and survives. This is the one lever that reaches reUSD's real backing, the harm our feed can only detect, not prevent. Secondary: delay the GuardianUpgradeable upgrade grant; nice-to-have since FiRM's independent feed covers the oracle-repoint harm. Note an AuthHook delay is not a config toggle; Core's hook is a synchronous gate, so a true delay means new trusted code. Either path is itself an 8-day Voter action. Internal: monitor the tripwire events in Monitoring Surface. |
| **3a. Security Audits** | 🟢 **PASS** | Six post-exploit engagements across ChainSecurity and Electisec with zero unresolved critical or high findings; direct exploit-fix review came back clean. Incumbent-only firm coverage noted. | - |
| **3b. Bug Bounty** | 🔴 **FAIL** | No active, well-defined bug bounty program located this cycle. Framework baseline requires an active bounty proportional to TVL. | Ask: confirm bounty host, scope, and maximum payout; if none exists, stand one up before any FiRM listing. |
| **4. Scope Stability** | 🟢 **PASS** | Rescan delta clean (no role, parameter, contract, or finding changes between consecutive scans). Contract set stable since the March 2026 Guardian upgrade. | Internal: weekly rescan continues; grade drift at reassessment. |
| **5. Mint / Redemption Halt** | 🟠 **FLAG** | Pause freezes new mint only; repay, redeem, and liquidate always remain open, so collateral is never trapped. However the ops Safe can instantly disable the below-peg permissionless redemption backstop, and Core can raise the redemption fee to 100% on the slow path, weakening the peg-defense mechanism FiRM's feed indirectly relies on. | Ask: state the intended policy for `updateGuardSettings` use and consider bounding the redemption fee below 100%. |
| **6. On-chain Liquidity** | 🟠 **FLAG** | Aggregate pairing depth roughly 3.13M USD; 2% single-swap depth roughly 136K USD on the best pool; 84% cumulative TVL decline from the April 2025 baseline; all pools roughly 3:1 reUSD-heavy and thinning week over week. Insufficient to support meaningful liquidation flow at any conventional market size. | Internal: any interim listing sized against pairing depth with a token supply ceiling. |
| **7. Protocol Maturity (Lindy)** | 🟢 **PASS** | 16.5 months on mainnet; 13.1 months since the June 2025 maturity-clock reset, clearing the 12-month standard. Remediation and recovery were completed within two months of the incident. | - |
| **8. On-chain Track Record** | 🟠 **FLAG** | One critical incident (9.5M USD, socialized), fully repaid by August 2025 with the insurance mechanism functioning as designed. Since then: clean governance record, no further incidents, but TVL down roughly 84% from the May 2025 peak and supply down 60%. Survival is proven; product-market recovery is not. | - |
| **9. Governance / Decentralization** | 🟠 **FLAG** | Fully on-chain governance with real participation (24 proposals, 30% quorum) is above peer standard; the effective-authority bypass in row 2 and RSUP vest concentration of 56.90% (43.072m is 56.90% of 75.693m) can temper it. | - |
| **10. Docs vs On-Chain** | 🟢 **PASS** | Documentation matches chain state on all load-bearing claims checked this cycle. Minor: PriceWatcher's internal oracle pointer is stale versus the live `REUSD_ORACLE` (scanner Q-07); cosmetic, does not affect the peg price. | - |
| **11. Freeze / Blocklist** | 🟢 **PASS** | No blocklist, freeze, or pausability on the reUSD token contract itself. Sanctions screening of the full role surface clean. | - |
| **12. Permissioning Profile** | 🟢 **PASS** | Zero EOA role holders across 138 role slots; all user paths permissionless; no whitelist gates anywhere in the user flow. | - |
| **13. Backing / Attestation** | 🟢 **PASS** | CDP stablecoin with backing fully on-chain and observable in real time (CurveLend and Fraxlend vault shares), which is stronger than any attestation regime. | - |

**Disposition mechanics.** A pre-screening does not assign parameters. FAIL rows carry remediation asks that must be answered or resolved before this asset advances to a full risk assessment; FLAG rows shape sizing and parameters at that stage. The recommended path is to deliver the remediation asks to the team, keep the asset on weekly Trustfall rescan and liquidity watch, and revisit when flags improve materially.

## Sources

PRIMARY: Trustfall Access Control Report, [Resupply USD (reUSD)](https://rwg-access-control-reports.vercel.app/reports/collateral/Resupply_reUSD.html), scan 2026-07-29 00:06 UTC, block 25634777 (K. / RWG internal).

On-chain reads at Ethereum block 25641571 via public RPC: reUSD totalSupply and balances, Insurance Pool, sreUSD vault, Curve pool balances, FiRM reUSD/USD feed and crvUSD/USD wrapper getters.

Curve prices API (prices.curve.finance), pool TVL and volume, July 29, 2026; DeFiLlama coins API for paired-asset prices; CoinGecko tickers (depth sampled July 21, 2026).

Resupply audits repository (github.com/resupplyfi/resupply/audits): ChainSecurity v2, ChainSecurity sreUSD, ChainSecurity CurveLend Operators, Electisec Update Report, Electisec sreUSD, Electisec CurveLend Operator.

Resupply governance forum, Savings reUSD (sreUSD) proposal; Resupply documentation (docs.resupply.finance).

QuillAudits Resupply hack analysis (June 2025 exploit technical reconstruction); Resupply recovery plan communications.

RWG, Complete Risk Assessment: reUSD/sDOLA LPT Collateral on FiRM (April 2025), including the 2025 feed, escrow, and market deployments.

FiRM Collateral Pre-Screening framework and the Re Protocol reUSD Pre-Screening v2 (July 7, 2026) as template.
