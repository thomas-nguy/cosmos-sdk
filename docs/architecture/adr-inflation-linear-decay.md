# ADR: Linear Inflation Decay (Coefficient-Based)

## Status

Draft

## Summary

Add an **optional** linear decay to the x/mint inflation rate, so that each block inflation decreases by a fixed amount (a **decay-per-block** coefficient) and tends to **zero**. This gives chains a simple, predictable way to reduce staking emissions over time.

## Context

The current mint module adjusts inflation each block based on the **bonded ratio** (distance from `GoalBonded`), with bounds `InflationMin` and `InflationMax`. There is no built-in notion of “inflation decreases over time” independent of bonding. Some chains want:

- A scheduled reduction in emissions each block regardless of bonded ratio.
- A clear, linear path from current inflation down to zero.

This ADR designs a **linear decay** applied on top of the existing bonded-ratio logic, parameterized by a **decay-per-block** coefficient (amount subtracted from the inflation rate each block).

## Design

### 1. Decay model

- **Linear per block:** Inflation is reduced by a fixed amount **each block**.
- **Coefficient:** `InflationDecayPerBlock` = amount subtracted from the inflation rate every block (e.g. a small decimal; magnitude depends on block time and desired annual effect).
- **Effective inflation** after decay (for this block):  
  `inflation_after_decay = max(0, inflation_before_decay - InflationDecayPerBlock)`  
  where `inflation_before_decay` is the result of the existing `NextInflationRate(params, bondedRatio)` (bonded-ratio adjustment, capped by `InflationMin`/`InflationMax`). Inflation is clamped at zero so it tends to zero and never goes negative.

So each block, inflation drops by exactly `InflationDecayPerBlock`, until it reaches zero. No conversion using `BlocksPerYear` is applied; the parameter is the per-block decay amount.

### 2. New parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `InflationDecayEnabled` | `bool` | If `true`, linear decay is applied each block; if `false`, behavior is unchanged (no decay). |
| `InflationDecayPerBlock` | `LegacyDec` | Amount subtracted from the inflation rate **each block** when decay is enabled. Must be ≥ 0. Inflation is clamped at zero so it tends to zero. |

- **Defaults (when decay is disabled):**  
  `InflationDecayEnabled = false`, `InflationDecayPerBlock = 0`. No change to current behavior.
- **Validation:**  
  `InflationDecayPerBlock >= 0`.

### 3. Order of operations (each block)

1. Compute **bonded-ratio-adjusted** inflation as today:  
   `inflation = NextInflationRate(params, bondedRatio)`  
   (already clamped to `[InflationMin, InflationMax]`).
2. If **not** `InflationDecayEnabled`, use this `inflation` as the new minter inflation and proceed to annual provisions / block provision as today.
3. If **`InflationDecayEnabled`**:
   - `inflation = max(0, inflation - params.InflationDecayPerBlock)`
   - Use this as the new minter inflation for the rest of the block (annual provisions, block provision).

So the **per-block coefficient** is the amount subtracted each block; inflation tends to **zero** and is never negative.

### 4. Formula: current vs after

**Current formula (no decay).** Each block the mint module computes the next inflation from the **bonded ratio** only:

1. **Rate change (per year, then scaled to per block):**
   \[
   \Delta_{\text{year}} = \left(1 - \frac{\text{bondedRatio}}{\text{GoalBonded}}\right) \times \text{InflationRateChange}
   \]
   \[
   \Delta_{\text{block}} = \frac{\Delta_{\text{year}}}{\text{BlocksPerYear}}
   \]

2. **Update and clamp:**
   \[
   \text{inflation}_{\text{next}} = \text{clamp}\big(\text{inflation}_{\text{current}} + \Delta_{\text{block}},\; \text{InflationMin},\; \text{InflationMax}\big)
   \]

So inflation moves toward a target implied by `GoalBonded` (e.g. 67% bonded), with max step `InflationRateChange/BlocksPerYear` per block, and is bounded by `[InflationMin, InflationMax]`. There is **no** time-based decay; the only change each block is this bonded-ratio adjustment.

**After formula (decay enabled).** The current formula is applied first; then, when `InflationDecayEnabled` is true, a **per-block decay** is applied and the result is clamped at zero:

\[
\text{inflation}_{\text{next}} = \max\left(0,\; \text{inflation}_{\text{next}} - \text{InflationDecayPerBlock}\right)
\]

So the full pipeline is: (1) bonded-ratio adjustment and clamp to `[InflationMin, InflationMax]`, then (2) subtract `InflationDecayPerBlock` and clamp at 0. Inflation therefore tends to zero over time when decay is enabled.

**Same in code form:**

- **Current:**  
  `inflationRateChangePerYear = (1 - bondedRatio/GoalBonded) * InflationRateChange`  
  `inflationRateChange = inflationRateChangePerYear / BlocksPerYear`  
  `inflation = clamp(currentInflation + inflationRateChange, InflationMin, InflationMax)`

- **After (with decay):**  
  `inflation = clamp(currentInflation + inflationRateChange, InflationMin, InflationMax)`  *(same as above)*  
  `inflation = max(0, inflation - InflationDecayPerBlock)`  *(extra step when decay enabled)*

**What changed in the formula:**

| Aspect | Before (current SDK) | After (this ADR, decay enabled) |
|--------|----------------------|----------------------------------|
| **Inflation used for minting** | `NextInflationRate(params, bondedRatio)` only, clamped to `[InflationMin, InflationMax]`. | Same, then subtract `InflationDecayPerBlock` and clamp **at 0** (not at a configurable floor). |
| **Lower bound** | `InflationMin` (e.g. 7%). | **Zero.** The floor is fixed at 0 so inflation tends to zero. |
| **Per-block change** | Only from bonded-ratio adjustment. | Bonded-ratio adjustment **minus** `InflationDecayPerBlock` (the decay amount per block). |
| **Over time** | No scheduled decrease. | Inflation decreases by `InflationDecayPerBlock` every block until it hits 0. |

So the only change to the effective formula is: **one extra step** that subtracts the per-block decay amount and then applies `max(0, ·)` so inflation never goes negative and tends to zero.

### 5. Formula summary (compact)

- **Per block:**  
  `inflation_after_decay = max(0, inflation_before_decay - InflationDecayPerBlock)`

So “reduce the inflation linearly per block” is implemented as: **subtract a fixed amount (`InflationDecayPerBlock`) from the inflation rate each block**, tending to zero.

### 6. Edge cases and notes

- **Zero:** Once inflation reaches zero, it stays at zero (bonded-ratio logic can still push it up within `[InflationMin, InflationMax]` next block; then decay will bring it back down again until it tends to zero).
- **Compatibility:** With `InflationDecayEnabled = false` (default), behavior is identical to current SDK. No migration required for existing chains.
- **Bonded-ratio vs decay:** Both can act in the same block: first bonded-ratio sets a candidate inflation; then decay subtracts a fixed amount. So the “target” from bonding is still respected in shape, but the baseline is shifted down over time toward zero.
- **Choosing the value:** Because decay is per block, the numeric value of `InflationDecayPerBlock` is typically small (e.g. if targeting ~2% annual reduction with 5s blocks, blocks per year ≈ 6.3M, so per block ≈ 0.02 / 6.3e6). Chains set it directly for the desired per-block step.

### 7. Optional: decay start height (future extension)

To start decay from a specific block (e.g. after a fork) instead of from genesis, one could add:

- `InflationDecayStartHeight` (int64): before this height, no decay; from this height onward, subtract `InflationDecayPerBlock` each block as above.

For this ADR we keep the design to “decay from first block when enabled”; the per-block coefficient is sufficient for the linear decay mechanism (goal: tend to zero).

## Where the code needs to be modified and what to change

The following lists every file to touch, the exact location, and the change. No other modules depend on mint `Params` in a way that requires changes; simulation and genesis use `DefaultParams()` / `Validate()`, so once types and keeper are updated they pick up the new fields.

---

### 1. Proto: add new params

**File:** `proto/cosmos/mint/v1beta1/mint.proto`

**Where:** Inside the `Params` message, after `blocks_per_year = 6;` (around line 61).

**Change:** Add two new fields:

```protobuf
  // when true, inflation is reduced each block by a fixed amount (tending to zero)
  bool inflation_decay_enabled = 7;
  // amount subtracted from the inflation rate each block when decay is enabled (per-block decay)
  string inflation_decay_per_block = 8 [
    (cosmos_proto.scalar)  = "cosmos.Dec",
    (gogoproto.customtype) = "cosmossdk.io/math.LegacyDec",
    (gogoproto.nullable)   = false,
    (amino.dont_omitempty) = true
  ];
}
```

**After editing:** Run the SDK proto code generator so that `x/mint/types/mint.pb.go` is regenerated (`make proto-gen`). The generated `Params` struct will then include `InflationDecayEnabled` and `InflationDecayPerBlock`; do not edit `mint.pb.go` by hand.

---

### 2. Types: defaults, constructor, validation

**File:** `x/mint/types/params.go`

**Changes:**

- **NewParams:** Add two parameters `inflationDecayEnabled bool`, `inflationDecayPerBlock sdkmath.LegacyDec` and set them on the returned `Params` struct.
- **DefaultParams:** In the literal `Params{ ... }`, set `InflationDecayEnabled: false`, `InflationDecayPerBlock: sdkmath.LegacyZeroDec()` (so existing behavior is unchanged).
- **Validate:** After existing checks, if `p.InflationDecayEnabled` is true, validate `p.InflationDecayPerBlock` (not nil, not negative). Add a helper `validateInflationDecayPerBlock(sdkmath.LegacyDec) error` and call it from `Validate()` when decay is enabled.

---

### 3. Keeper: apply decay after computing inflation

**File:** `x/mint/keeper/mint.go`

**Where:** Inside `DefaultMintFn`, immediately after `minter.Inflation = ic(ctx, minter, params, bondedRatio)` and before `minter.AnnualProvisions = ...`.

**Change:** Insert the decay step when enabled:

1. After `minter.Inflation = ic(ctx, minter, params, bondedRatio)`:
   - If `params.InflationDecayEnabled`:
     - Set `minter.Inflation = minter.Inflation.Sub(params.InflationDecayPerBlock)`.
     - If `minter.Inflation.LT(math.LegacyZeroDec())` then set `minter.Inflation = math.LegacyZeroDec()`.
2. Leave the rest unchanged: `minter.AnnualProvisions = minter.NextAnnualProvisions(...)`, minting, events, etc.

**Import:** Ensure `cosmossdk.io/math` is imported in `mint.go` if needed for `math.LegacyZeroDec()` (use the same style as the rest of the file).

---

### 4. Call sites that build or validate Params

- **Genesis:** `x/mint/types/genesis.go` uses `DefaultParams()` and `Params.Validate()`. No change needed once `DefaultParams()` and `Validate()` include the new fields.
- **Simulation:** `x/mint/simulation/proposals.go` and `msg_factory.go` use `types.DefaultParams()`. No change needed; new fields will be zero/false by default.
- **Tests:** `x/mint/types/params_test.go` and any test that builds `Params` with `NewParams(...)` must be updated to pass the two new arguments; add tests for `validateInflationDecayPerBlock` and for decay logic in the keeper when enabled.

---

### 5. Summary table

| File | Modification |
|------|--------------|
| `proto/cosmos/mint/v1beta1/mint.proto` | Add `inflation_decay_enabled` (bool) and `inflation_decay_per_block` (Dec) to `Params`. |
| Regenerated `x/mint/types/mint.pb.go` | Done by `make proto-gen`; do not edit by hand. |
| `x/mint/types/params.go` | Extend `NewParams`, `DefaultParams` (decay disabled, per-block 0), `Validate()` (validate per-block when enabled); add `validateInflationDecayPerBlock`. |
| `x/mint/keeper/mint.go` | After `minter.Inflation = ic(...)`, if `params.InflationDecayEnabled` then subtract `params.InflationDecayPerBlock` and clamp at zero (no division by BlocksPerYear). |
| Tests | Update `NewParams` call sites; add tests for new validation and decay behavior. |

No change to Minter state shape; decay is stateless.


## References

- Current inflation logic: `x/mint/types/minter.go` (`NextInflationRate`), `x/mint/keeper/mint.go` (`DefaultMintFn`).
- Params: `x/mint/types/params.go`, `proto/cosmos/mint/v1beta1/mint.proto`.
