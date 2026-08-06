# FiRM reUSD/sDOLA collateral price: stress backtest

## What FiRM books, and the three channels that move it

Verified live on-chain. `CurveLPPessimisticFeed 0x62cb0c8fe7d9e026f4e16451b1318c03d874cf1d` (`description() = "reusdsdola / USD"`):

```
LP_price = min(reUSD/USD, DOLA=1.0) x reUSD_sDOLA_pool.get_virtual_price()
reUSD/USD = crvUSD/USD / reUSD_scrvUSD_pool.price_oracle(0)
```

Arithmetic confirmed at the tip: `min(0.99020970, 1.0) x 1.07929966 = 1.06873299`, equal to `latestAnswer()`. reUSD/USD never exceeded 0.9961 in 42,643 samples, so the pessimistic `min` always selects reUSD and the DOLA leg is never binding.

| channel | behaviour | single-block risk |
|---|---|---|
| crvUSD/USD Chainlink wrapper `0x237C421F` | step function, 9-31 discrete updates per window | **yes** - passes through 1:1, measured gain 1.000000 (max deviation 4.5e-14) |
| reUSD/scrvUSD EMA `price_oracle(0)` | `ma_exp_time()=866 s`, **10.0 min half-life** | no - bounded per block, but **lags genuine moves** |
| sDOLA price-per-share via `get_virtual_price()` | **unsmoothed spot** | **yes** - +13.79% in one block on 2026-03-02 (exploit) |

Because `log(feed) = log(crvUSD) - log(EMA)`, the two reUSD legs are **additive in log space, not multiplicative**: a crvUSD deviation is passed through at unity gain and is not amplified by the EMA. They compound only in the sense that simultaneous adverse moves sum. The crvUSD aggregator's circuit breakers are inert (`minAnswer=1e-8`, `maxAnswer=9.6e44`), so pass-through is unbounded.

> Redemption context (not a CF input): `permissionlessPriceThreshold() = 0.985` in **crvUSD** terms, `baseRedemptionFee() = 1%`. The floor was never approached - closest was Event B at +10.6 bps above it, with 0 samples below in any window.

---

## Event A - Resupply exploit (Jun 2025)

- Blocks 22770564..22856445, 13024 samples (13024 expected, 0 missing), 288.0 h.
- UTC 2025-06-24 00:00:11 .. 2025-07-06 00:00:11

### Collateral price (what FiRM books) - drawdown

Drawdown, not range: range includes upward virtual-price drift, which never threatens a position.

| horizon | max adverse drawdown | max favorable |
|---|---|---|
| 5m | *(600 s grain cannot resolve 5m)* | |
| 30m | **0.0929%** | 0.1326% |
| 1h | **0.1310%** | 0.1937% |
| 6h | **0.1861%** | 0.2443% |
| 24h | **0.2774%** | 0.2898% |
| full window | **0.2774%** | |

### Single-block (12 s) moves

- **collateral LP price**: worst single-block -0.0322% (blk 22785462, 2025-06-26 01:54:11); largest up +0.0038% (blk 22792037, 2025-06-26 23:56:23)
- **reUSD/USD feed**: worst single-block -0.0336% (blk 22785462, 2025-06-26 01:54:11); largest up +0.0023% (blk 22786041, 2025-06-26 03:50:47)
- **crvUSD/USD leg**: worst single-block -0.0252% (blk 22791851, 2025-06-26 23:18:59); largest up +0.0000%
- **EMA leg**: worst single-block -0.0023% (blk 22786041, 2025-06-26 03:50:47); largest up +0.0336% (blk 22785462, 2025-06-26 01:54:11)
- **sDOLA price-per-share**: worst single-block +0.0000%; largest up +0.0000% (blk 22794927, 2025-06-27 09:37:11)

### EMA lag: how far the booked price sat above the pool's instantaneous price

`feed = crvUSD/EMA` and the spot-implied value is `crvUSD/spot`, so the overpricing is exactly `spot/EMA - 1`. Positive means FiRM books reUSD **higher** than the pool instantaneously implies.

| measure | value |
|---|---|
| max **overpricing** (EMA below spot) | **+2.4386%** at blk 22785461, 2025-06-26 01:53:59 |
| max underpricing (EMA above spot) | -0.1505% at blk 22785985 |
| max absolute EMA-vs-spot divergence | 2.4386% |
| samples with any overpricing | 6228 of 13024 (47.8%) |
| mean overpricing while positive | 0.0096% |

**Persistence** (consecutive-block region only, so durations are real):

| overpricing exceeds | total blocks | wall time | longest unbroken run |
|---|---|---|---|
| 0.25% | 4 | 0.8 min | 4 blocks (0.8 min) |
| 0.50% | 3 | 0.6 min | 3 blocks (0.6 min) |
| 1.00% | 2 | 0.4 min | 2 blocks (0.4 min) |
| 2.00% | 2 | 0.4 min | 2 blocks (0.4 min) |

### sDOLA price-per-share (unsmoothed component of the virtual price)

- Range 1.136556 -> 1.139295 over the window (+0.241%).
- No single-block decline beyond -0.05% observed in this window.

### Levels relative to par

- reUSD/USD: min 0.989076, max 0.992153, mean **101.5 bps below $1.00** the whole window.
- Collateral LP price: min 0.993752, max 0.997099.
- EMA-source pool depth: 93,094,092 -> 50,360,474 token units.
- reUSD share of that pool: 77.2% -> 77.2% (all-sample max 87.4%, grain-matched 600 s max 80.0%).

---

## Event B - LlamaLend sDOLA-long2 exploit (Mar 2026)

- Blocks 24551712..24616188, 12596 samples (12596 expected, 0 missing), 216.0 h.
- UTC 2026-02-28 00:00:11 .. 2026-03-09 00:00:11

### Collateral price (what FiRM books) - drawdown

Drawdown, not range: range includes upward virtual-price drift, which never threatens a position.

| horizon | max adverse drawdown | max favorable |
|---|---|---|
| 5m | *(600 s grain cannot resolve 5m)* | |
| 30m | **0.1677%** | 3.7326% |
| 1h | **0.3081%** | 3.7331% |
| 6h | **0.4339%** | 4.2946% |
| 24h | **0.4553%** | 4.4724% |
| full window | **0.4553%** | |

### Single-block (12 s) moves

- **collateral LP price**: worst single-block -0.0040% (blk 24568799, 2026-03-02 09:15:11); largest up +3.7294% (blk 24566937, 2026-03-02 03:00:11)
- **reUSD/USD feed**: worst single-block -0.0040% (blk 24568799, 2026-03-02 09:15:11); largest up +0.1498% (blk 24571001, 2026-03-02 16:37:59)
- **crvUSD/USD leg**: worst single-block +0.0000%; largest up +0.1498% (blk 24571001, 2026-03-02 16:37:59)
- **EMA leg**: worst single-block -0.0163% (blk 24568583, 2026-03-02 08:31:47); largest up +0.0040% (blk 24568799, 2026-03-02 09:15:11)
- **sDOLA price-per-share**: worst single-block +0.0000%; largest up +13.7946% (blk 24566937, 2026-03-02 03:00:11)

### EMA lag: how far the booked price sat above the pool's instantaneous price

`feed = crvUSD/EMA` and the spot-implied value is `crvUSD/spot`, so the overpricing is exactly `spot/EMA - 1`. Positive means FiRM books reUSD **higher** than the pool instantaneously implies.

| measure | value |
|---|---|
| max **overpricing** (EMA below spot) | **+0.1641%** at blk 24571233, 2026-03-02 17:24:35 |
| max underpricing (EMA above spot) | -0.9912% at blk 24568558 |
| max absolute EMA-vs-spot divergence | 0.9912% |
| samples with any overpricing | 5888 of 12596 (46.7%) |
| mean overpricing while positive | 0.0123% |

**Persistence** (consecutive-block region only, so durations are real):

| overpricing exceeds | total blocks | wall time | longest unbroken run |
|---|---|---|---|
| 0.25% | 0 | never | - |
| 0.50% | 0 | never | - |
| 1.00% | 0 | never | - |
| 2.00% | 0 | never | - |

### sDOLA price-per-share (unsmoothed component of the virtual price)

- Range 1.188756 -> 1.361370 over the window (+14.521%).
- **Discontinuity: +13.7946% in a single block** at blk 24566937 (2026-03-02 03:00:11).
- No single-block decline beyond -0.05% observed in this window.

### Levels relative to par

- reUSD/USD: min 0.986041, max 0.995005, mean **106.5 bps below $1.00** the whole window.
- Collateral LP price: min 1.005074, max 1.054006.
- EMA-source pool depth: 9,795,684 -> 9,577,994 token units.
- reUSD share of that pool: 77.8% -> 71.8% (all-sample max 79.1%, grain-matched 600 s max 79.1%).

---

## Baseline - calm control (May 2026)

- Blocks 24996368..25060965, 1293 samples (1293 expected, 0 missing), 216.0 h.
- UTC 2026-05-01 00:00:11 .. 2026-05-10 00:00:11

### Collateral price (what FiRM books) - drawdown

Drawdown, not range: range includes upward virtual-price drift, which never threatens a position.

| horizon | max adverse drawdown | max favorable |
|---|---|---|
| 5m | *(600 s grain cannot resolve 5m)* | |
| 30m | **0.2418%** | 0.0494% |
| 1h | **0.2701%** | 0.0572% |
| 6h | **0.2810%** | 0.0623% |
| 24h | **0.2810%** | 0.1321% |
| full window | **0.3323%** | |

### Single-block (12 s) moves

- No consecutive-block sampling in this window (coarse grain only), so no single-block figure can be produced.

### EMA lag: how far the booked price sat above the pool's instantaneous price

`feed = crvUSD/EMA` and the spot-implied value is `crvUSD/spot`, so the overpricing is exactly `spot/EMA - 1`. Positive means FiRM books reUSD **higher** than the pool instantaneously implies.

| measure | value |
|---|---|
| max **overpricing** (EMA below spot) | **+0.1507%** at blk 25059418, 2026-05-09 18:49:23 |
| max underpricing (EMA above spot) | -0.0387% at blk 25006368 |
| max absolute EMA-vs-spot divergence | 0.1507% |
| samples with any overpricing | 223 of 1293 (17.2%) |
| mean overpricing while positive | 0.0021% |

### sDOLA price-per-share (unsmoothed component of the virtual price)

- Range 1.376029 -> 1.379305 over the window (+0.238%).
- No single-block decline beyond -0.05% observed in this window.

### Levels relative to par

- reUSD/USD: min 0.992320, max 0.996047, mean **45.5 bps below $1.00** the whole window.
- Collateral LP price: min 1.060849, max 1.064386.
- EMA-source pool depth: 7,899,231 -> 7,810,813 token units.
- reUSD share of that pool: 70.8% -> 74.6% (all-sample max 74.6%, grain-matched 600 s max 74.6%).

---

## Recent - current thin-pool regime (Jul-Aug 2026)

- Blocks 25469764..25690981, 15730 samples (15730 expected, 0 missing), 740.0 h.
- UTC 2026-07-06 00:00:11 .. 2026-08-05 20:00:59

### Collateral price (what FiRM books) - drawdown

Drawdown, not range: range includes upward virtual-price drift, which never threatens a position.

| horizon | max adverse drawdown | max favorable |
|---|---|---|
| 5m | *(600 s grain cannot resolve 5m)* | |
| 30m | **0.0908%** | 0.1017% |
| 1h | **0.0997%** | 0.1205% |
| 6h | **0.1301%** | 0.1279% |
| 24h | **0.1696%** | 0.1345% |
| full window | **0.3243%** | |

### Single-block (12 s) moves

- **collateral LP price**: worst single-block -0.0310% (blk 25690220, 2026-08-05 17:28:47); largest up +0.0102% (blk 25683049, 2026-08-04 17:28:35)
- **reUSD/USD feed**: worst single-block -0.0310% (blk 25690220, 2026-08-05 17:28:47); largest up +0.0102% (blk 25683049, 2026-08-04 17:28:35)
- **crvUSD/USD leg**: worst single-block -0.0310% (blk 25690220, 2026-08-05 17:28:47); largest up +0.0102% (blk 25683049, 2026-08-04 17:28:35)
- **EMA leg**: worst single-block -0.0007% (blk 25686024, 2026-08-05 03:25:23); largest up +0.0037% (blk 25684652, 2026-08-04 22:50:35)
- **sDOLA price-per-share**: worst single-block +0.0000%; largest up +0.0000% (blk 25686995, 2026-08-05 06:40:35)

### EMA lag: how far the booked price sat above the pool's instantaneous price

`feed = crvUSD/EMA` and the spot-implied value is `crvUSD/spot`, so the overpricing is exactly `spot/EMA - 1`. Positive means FiRM books reUSD **higher** than the pool instantaneously implies.

| measure | value |
|---|---|
| max **overpricing** (EMA below spot) | **+0.1475%** at blk 25684644, 2026-08-04 22:48:47 |
| max underpricing (EMA above spot) | -0.0633% at blk 25653064 |
| max absolute EMA-vs-spot divergence | 0.1475% |
| samples with any overpricing | 3715 of 15730 (23.6%) |
| mean overpricing while positive | 0.0032% |

**Persistence** (consecutive-block region only, so durations are real):

| overpricing exceeds | total blocks | wall time | longest unbroken run |
|---|---|---|---|
| 0.25% | 0 | never | - |
| 0.50% | 0 | never | - |
| 1.00% | 0 | never | - |
| 2.00% | 0 | never | - |

### sDOLA price-per-share (unsmoothed component of the virtual price)

- Range 1.397003 -> 1.406633 over the window (+0.689%).
- No single-block decline beyond -0.05% observed in this window.

### Levels relative to par

- reUSD/USD: min 0.990949, max 0.995856, mean **76.6 bps below $1.00** the whole window.
- Collateral LP price: min 1.069195, max 1.072970.
- EMA-source pool depth: 6,853,251 -> 7,605,410 token units.
- reUSD share of that pool: 69.2% -> 75.6% (all-sample max 77.2%, grain-matched 600 s max 77.2%).

---

## Cross-window comparison

Rows marked *(600 s)* use the coarse grain, the only one present in every window.

| statistic | Event A (Resupply) | Event B (LlamaLend) | Baseline (calm) | Recent (thin pool) |
|---|---|---|---|---|
| samples | 13024 | 12596 | 1293 | 15730 |
| **collateral LP drawdown, 24h** *(600 s)* | **0.2774%** | **0.4553%** | **0.2810%** | **0.1696%** |
| collateral LP drawdown, 1h *(600 s)* | 0.1310% | 0.3081% | 0.2701% | 0.0997% |
| collateral LP drawdown, full window | 0.2774% | 0.4553% | 0.3323% | 0.3243% |
| worst single-block LP move | -0.0322% | -0.0040% | n/a | -0.0310% |
| largest single-block LP rise | +0.0038% | +3.7294% | n/a | +0.0102% |
| **max EMA overpricing vs spot** | **+2.4386%** | **+0.1641%** | **+0.1507%** | **+0.1475%** |
| max EMA-vs-spot divergence (abs) | 2.4386% | 0.9912% | 0.1507% | 0.1475% |
| sDOLA PPS largest single-block rise | +0.0000% | +13.7946% | n/a | +0.0000% |
| reUSD/USD mean below par | 101.5 bps | 106.5 bps | 45.5 bps | 76.6 bps |
| EMA-source pool depth (end) | 50,360,474 | 9,577,994 | 7,810,813 | 7,605,410 |
| reUSD share max (600 s grain) | 80.0% | 79.1% | 74.6% | 77.2% |

---

## EMA lag, in closed form

`ma_exp_time() = 866 s` on the reUSD/scrvUSD pool, i.e. a **10.0 minute half-life**. After an instantaneous genuine decline of D%, the residual overpricing decays as `D x exp(-t/866)`:

| elapsed | residual factor | overpricing if reUSD truly fell 10% |
|---|---|---|
| 1 min | 0.9331 | 9.33% |
| 5 min | 0.7072 | 7.07% |
| 10 min | 0.5002 | 5.00% |
| 30 min | 0.1251 | 1.25% |
| 1 h | 0.0157 | 0.16% |
| 6 h | 0.0000 | 0.00% |

The smoothing is symmetric: it bounds single-block manipulation of the reUSD leg, and by the same mechanism it delays recognition of a genuine collapse. It does **not** smooth the sDOLA channel, which enters the collateral price unsmoothed through `get_virtual_price()`.

**What the windows can and cannot tell us about this.** The largest EMA-vs-spot gap observed anywhere is +2.4386% in Event A, but it persisted for **2 blocks (24 s)** and sat inside the same 3-block transient that produced the 87.4% all-sample reUSD share reading. That is the EMA correctly rejecting a momentary spot excursion, which is the mechanism working, not failing. Sustained overpricing above 0.25% never lasted beyond 0.8 minutes in any window, and in Events B and Recent it never occurred at all.

Consequently the lag risk is **untested rather than disproven**: none of the four windows contains a genuine, sustained reUSD collapse for the EMA to have lagged. The largest sustained collateral drawdown in the entire dataset is 0.4553% over 24 h. The closed-form table above is therefore the only available estimate of what the lag would cost in a real failure, and it is a projection, not an observation.

---

## Collateral pool depth: the binding constraint

The reUSD/sDOLA pool that *is* the collateral has fallen **99.5%** (read directly per block, not from the sampled series):

| date | LP totalSupply | balances (reUSD / sDOLA) |
|---|---|---|
| 2025-06 (Event A) | 3,950,332 | 2,968,006 / 883,140 |
| 2026-03 (Event B) | 234,313 | 174,670 / 54,104 |
| 2026-05 (baseline) | 77,119 | 33,939 / 35,164 |
| **2026-08 (now)** | **20,886** | **16,103** / **4,586** |

Current pool TVL is approximately **22,394 USD**. Note this is the *collateral* pool; the reUSD/USD price feed is driven by the separate reUSD/scrvUSD pool (7.6M) plus Chainlink, so feed quality and collateral depth are distinct questions. At 22k of underlying liquidity, the constraint on any market is position size and liquidation feasibility rather than the choice of collateral factor.

## Scope note: the sDOLA channel

The collateral price contains an unsmoothed sDOLA component (`get_virtual_price()` reads sDOLA price-per-share live), which produced the single-block **+3.73%** rise recorded in Event B during the 2026-03-02 LlamaLend exploit. That channel was investigated separately and found **not** to be a lending risk here: sDOLA price-per-share is monotonically non-decreasing (200 samples over ~304 days, zero decreases; `MAX_ASSETS` is an immutable constant; staked principal is protected from governance itself), so there is no downward leg for a borrow-and-revert attack, and an inflation moves FiRM positions to healthier rather than unhealthier. It is recorded only so the +3.73% step in the Event B series is not mistaken for price risk.

**The collateral-factor question is therefore driven by the reUSD leg** - the one exogenous component that can actually fall, and the one subject to the 10-minute EMA lag documented above.
