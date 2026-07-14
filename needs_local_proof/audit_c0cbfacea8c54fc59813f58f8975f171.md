<!-- deepwiki_source_url: https://deepwiki.com/search/-deepwiki-candidate-triage-pro_29abb763-cce5-4741-af8d-90ccac541ce5?mode=deep -->
<!-- deepwiki_verdict: needs_local_proof -->

## Verdict
NEEDS_LOCAL_PROOF

## Paid Scope Match
fund_extraction

## Exact Code Path

**file:** `src/contracts/src/library/OracleLibrary.sol`
**function:** `getOldestObservationSecondsAgo`
**symbols/lines:** lines 116–141

```
(observationIndex + 1) % observationCardinality  // line 132
// when cardinality=1: always == 0, reads observations[0] (most recent slot)
secondsAgo = uint32(block.timestamp) - observationTimestamp;  // line 140
```

**file:** `src/contracts/src/TorusBuyAndProcess.sol`
**function:** `getTorusQuoteForTitanX` (lines 1120–1140), `_swapTitanXForTorus` (lines 821–840)
**symbols/lines:**

```
uint32 oldestObservation = OracleLibrary.getOldestObservationSecondsAgo(poolAddress);  // line 1125
if (oldestObservation < secondsAgo) secondsAgo = oldestObservation;  // line 1128
// collapses secondsAgo to near-zero when cardinality=1
uint256 adjustedTorusAmount = (expectedTorusAmount * (100 - titanXToTorusSlippage)) / 100;  // line 829
amountOutMinimum: adjustedTorusAmount  // line 837
```

## Attacker Path

**preconditions:**
- The TORUS/TitanX Uniswap V3 pool has `observationCardinality=1` (never increased beyond pool initialization — the protocol's `addLiquidityToTorusTitanXPool` does not call `increaseObservationCardinalityNext`)
- The pool has had at least one swap so `observations[0].timestamp` is set to a recent block
- The attacker has sufficient TitanX capital to move the pool price meaningfully

**attacker-controlled inputs:**
- Timing of a large TitanX→TORUS swap in block N-1 to manipulate spot price and update `observations[0].timestamp` to block N-1
- Submission of `swapTitanXForTorusAndBurn` victim tx in block N (or MEV-sandwich ordering)

**call sequence:**
1. Block N-1: Attacker swaps large TitanX→TORUS, moving spot price up (inflating TORUS price in TitanX terms). This writes `observations[0].timestamp = block.timestamp(N-1)`.
2. Block N: `getOldestObservationSecondsAgo` returns `block.timestamp(N) - block.timestamp(N-1) ≈ 12 seconds`. `secondsAgo` is clamped to ~12 instead of 900.
3. Block N: `consult(pool, 12)` returns a tick reflecting the manipulated price from block N-1.
4. Block N: `getTorusQuoteForTitanX` returns an inflated TORUS quote → `adjustedTorusAmount` is inflated → `amountOutMinimum` is set high relative to fair value but the actual swap still executes at the manipulated spot.

Wait — the direction matters. If the attacker inflates TORUS price (fewer TORUS per TitanX), `amountOutMinimum` is *lower* than fair, allowing the protocol swap to execute at a worse rate. Attacker front-runs by buying TORUS (raising its price in TitanX), protocol swap gets fewer TORUS at the inflated price, attacker back-runs selling TORUS.

5. Block N: Protocol's `swapTitanXForTorusAndBurn` executes with a low `amountOutMinimum` (oracle reflects manipulated price), swap fills at the manipulated rate.
6. Block N: Attacker back-runs, selling TORUS back to restore price, capturing the spread.

**Note:** Same-block manipulation is self-defeating: if the attacker's front-run swap occurs in the same block, `observations[0].timestamp = block.timestamp`, making `secondsAgo = 0`, which causes `consult` to revert with `"BP"`. The attack therefore requires a multi-block setup (manipulate in N-1, exploit in N), which is feasible via MEV but adds cost.

## Why Existing Checks Fail [1](#0-0) 

The `require(observationCardinality > 0, "NI")` check only guards against uninitialized pools. It does not guard against cardinality=1. When cardinality=1, `(observationIndex+1) % 1 == 0` always, so the function always reads `observations[0]` — the most recently written slot — regardless of `observationIndex`. This makes the "oldest observation" identical to the newest, collapsing the window. [2](#0-1) 

The fallback `if (oldestObservation < secondsAgo) secondsAgo = oldestObservation` is intended to handle young pools, but it has no floor. There is no `require(secondsAgo >= MIN_TWAP_SECONDS)` or equivalent. The only floor is the implicit `require(secondsAgo != 0, "BP")` inside `consult`, which only prevents the zero case, not the 1–12 second case. [3](#0-2) 

The `amountOutMinimum` is derived entirely from the oracle quote. With a collapsed TWAP window, the quote reflects the manipulated spot, and the slippage tolerance (`titanXToTorusSlippage`, default 10%) does not compensate for a large price manipulation.

The protocol never calls `increaseObservationCardinalityNext` on the TORUS/TitanX pool it creates, so cardinality remains 1 unless a third party increases it.

## Rejection Checks

**expected behavior checked:** The 15-minute TWAP fallback to `oldestObservation` is intended for young pools, but cardinality=1 is a permanent structural condition, not a transient startup state. This is not expected behavior — it is a design gap.

**prior report checked:** No evidence of a prior report in the indexed codebase.

**README/NatSpec checked:** No NatSpec or documentation acknowledges cardinality=1 as an accepted risk or expected state.

**unsupported assumption checked:** The TITAN_X_ETH_POOL (`0xc45a81bc23a64ea556ab4cdf08a86b61cdceea8b`) is an established external pool with high cardinality — the `getTitanXQuoteForETH` path is not vulnerable. The vulnerability is scoped to the TORUS/TitanX pool created by the protocol, which starts at cardinality=1 and has no code path to increase it.

## Local Proof Required

**test type:** Foundry fork test (mainnet fork, pinned block)

**test file to add:** `src/contracts/test/OracleCardinalityExploit.t.sol`

**test setup:**
1. Fork mainnet at a recent block.
2. Deploy or locate the TORUS/TitanX pool; confirm `slot0().observationCardinality == 1` via `IUniswapV3Pool(pool).slot0()`.
3. If cardinality > 1 on mainnet, use `vm.store` to set the cardinality field in `slot0` to 1 and set `observations[0].timestamp` to `block.timestamp - 12`.
4. Record `getTorusQuoteForTitanX(1e18)` as the baseline fair quote.
5. Simulate attacker front-run: large TitanX→TORUS swap to move price 10x, then `vm.roll(block.number + 1)` / `vm.warp(block.timestamp + 12)`.
6. Call `getTorusQuoteForTitanX(1e18)` again; record manipulated quote.
7. Call `swapTitanXForTorusAndBurn` and observe actual TORUS burned.

**expected assertion:**
```solidity
// Manipulated quote deviates from fair TWAP by >5%
assertGt(fairQuote, manipulatedQuote * 105 / 100, "oracle not collapsed");
// OR: actual TORUS burned is less than fair-price equivalent
assertLt(torusBurned, fairTorusBurned * 95 / 100, "sandwich not profitable");
```

**failure condition:** If the TORUS/TitanX pool on mainnet already has `observationCardinality > 1` (e.g., increased by a third party), the attack is not currently exploitable and the finding degrades to a latent risk for newly deployed pools. Confirm live cardinality via `cast call 0x<pool> "slot0()(uint160,int24,uint16,uint16,uint16,uint8,bool)" --rpc-url mainnet` and check the 4th return value.

### Citations

**File:** src/contracts/src/library/OracleLibrary.sol (L128-141)
```text
        require(observationCardinality > 0, "NI");

        (uint32 observationTimestamp, , , bool initialized) = IUniswapV3Pool(
            pool
        ).observations((observationIndex + 1) % observationCardinality);

        // The next index might not be initialized if the cardinality is in the process of increasing
        // In this case the oldest observation is always in index 0
        if (!initialized) {
            (observationTimestamp, , , ) = IUniswapV3Pool(pool).observations(0);
        }

        secondsAgo = uint32(block.timestamp) - observationTimestamp;
    }
```

**File:** src/contracts/src/TorusBuyAndProcess.sol (L828-837)
```text
        uint256 expectedTorusAmount = getTorusQuoteForTitanX(amountTitanX);
        uint256 adjustedTorusAmount = (expectedTorusAmount * (100 - titanXToTorusSlippage)) / 100;

        ISwapRouter.ExactInputParams memory params = ISwapRouter
            .ExactInputParams({
                path: path,
                recipient: address(this),
                deadline: _deadline,
                amountIn: amountTitanX,
                amountOutMinimum: adjustedTorusAmount
```

**File:** src/contracts/src/TorusBuyAndProcess.sol (L1124-1132)
```text
        uint32 secondsAgo = 15 * 60;
        uint32 oldestObservation = OracleLibrary.getOldestObservationSecondsAgo(
            poolAddress
        );
        if (oldestObservation < secondsAgo) secondsAgo = oldestObservation;
        (int24 arithmeticMeanTick, ) = OracleLibrary.consult(
            poolAddress,
            secondsAgo
        );
```
