# Architecture: Tiered Lock Rewards (Tier Logic)

## 1. Overview

### 1.1 Goal

Allow users to **lock tokens into a tier** (e.g. 1, 2, or 5 years). Locked tokens are **internal to the tier mechanism**: they can only be delegated or redelegated **within** the tier module (undelegation only after trigger exit); (like internal liquid-stake tokens that cannot be used outside this system). **To receive any rewards, users must delegate their position to a validator** — this requirement ensures tier lockers provide security to the network. Users receive:

- **Base rewards** from normal staking (block rewards, fees) **only when** their tier-locked tokens are delegated to validators.
- **Bonus rewards** as a **fixed APY** on the locked amount, paid from an **external rewards pool**, **only when** the position is delegated to a validator.

The tier module holds the locked tokens and is the delegator in `x/staking`; users operate via tier-specific messages (TierDelegate, TierRedelegate; **TierUndelegate is only allowed after the user has triggered exit** — tier lockers cannot voluntarily undelegate while in the tier, so they keep providing security until they leave). **Add to position:** the owner can **add tokens to an existing position** (same `position_id`) as long as exit has **not** been triggered. **Withdraw from tier:** the user can **trigger exit at any time**. Once they trigger exit, an **exit commitment** starts (wait X years, depending on tier). **Once that commitment has elapsed**, no more bonus is paid and the user can claim their tokens (after unbonding if delegated). To unbond, the user must have triggered exit and then may call **MsgTierUndelegate**.

### 1.2 Non-Goals

- Changing how base delegation rewards are computed or distributed.
- Replacing or duplicating `x/distribution`; we integrate with it.
- Managing inflation or chain-level reward issuance; the external pool is filled by external logic (governance, grants, another module).

---

## 2. Concepts

| Term | Description |
|------|-------------|
| **Tier** | A lock level defined by an **exit commitment duration** (e.g. 1y, 2y, 5y wait after user triggers exit), a **fixed bonus APY** (e.g. 0.10 = 10% per year on the locked amount), and a **minimum lock amount** (smallest amount that can be locked when creating a position in this tier). |
| **Tier-locked tokens** | Tokens sent to the tier module when locking; they are **internal** to the mechanism. They can only be delegated/redelegated via tier messages; **undelegation is only allowed after the owner has triggered exit** (internal liquid-stake style). They cannot be used externally (no transfer out except via withdraw from tier). |
| **Lock** | User sends tokens to the tier module and receives a **tier position**. The locked amount earns **no rewards** until the owner delegates to a validator; once delegated, it earns base (staking) rewards and a **fixed APY** bonus from the pool. **Tier lockers cannot undelegate** until they have triggered exit (they must stay delegated to provide security). User can trigger exit at any time. The owner can **add** to an existing position (same tier) as long as exit has not been triggered. |
| **Base rewards** | Staking rewards from `x/distribution` **only when** tier-locked tokens are delegated to validators (tier module is the delegator). No base rewards accrue while the position is not delegated. |
| **Bonus rewards** | **Fixed APY** on the locked (or delegated) amount, **only while the position is delegated** to a validator: `accrued_bonus = amount × BonusAPY × (time_elapsed / 1 year)`, paid from the **tier rewards pool**. No bonus accrues when the position is not delegated. |
| **Tier rewards pool** | Module account (or external keeper) holding coins used only to pay APY bonus. Filled by governance, grants, or another module. |
| **Tier position** | A record stored in module state: unique position ID, owner, tier_id, **amount_locked**, optional exit triggered time and exit unlock time, optional validator and **delegated shares** (if delegated). A position **cannot be broken down**: the full amount is delegated to a single validator as a whole (all-or-nothing). |
| **Withdraw from tier** | Two steps: (1) **Trigger exit** — user can trigger at any time. This starts the **exit commitment** (wait X years, depending on tier). (2) **Claim** — once the exit commitment has elapsed, no more bonus; user can claim tokens (after unbonding if delegated) and position is closed. |

---

## 3. High-Level Design

- **New module**: `x/tieredrewards` (or `x/lockrewards`).
- **Tier-locked tokens**: Users **lock tokens** into the tier module (transfer to module account). Those tokens are **internal liquid-stake style**: they can only be **delegated** or **redelegated** via the tier module’s own messages; undelegation is only allowed after the user has triggered exit (tier lockers cannot voluntarily undelegate while in the tier). They cannot be used outside this mechanism (no external LST).
- **Delegator in staking**: The **tier module account** is the delegator for all tier-locked delegations. Each position is delegated as a whole (full amount to one validator); the module stores per-position the validator address and the **delegated shares** returned by staking when delegating that amount. Base rewards are attributed to position owners when withdrawn. **No rewards (base or bonus) are paid unless the position is delegated** — lockers must delegate to a validator to earn rewards and thus provide security to the network.
- **Bonus**: **Fixed APY** on the locked amount (or on the delegated amount), accrued over time **only while the position is delegated**, and paid from the tier rewards pool. No multiplier on base rewards.
- **Exit commitment only**: User can trigger exit at any time. When they trigger exit, an exit commitment (X years, depending on tier) starts; once it has elapsed, no more bonus and they can claim tokens (after unbonding if delegated).
- **Dependencies**: `x/distribution`, `x/staking`, `x/bank`, `x/auth`. Tier module calls staking to delegate/undelegate/redelegate from its module account and distribution to withdraw rewards. So that **tier lockers can vote** in governance, the tier module exposes voting power per address and the app wires a custom gov tally that includes it (see **§8**).

---

## 4. State Model

### 4.1 Params (or stored config)

```go
// TierDefinition defines a single tier.
type TierDefinition struct {
    TierId                 uint32        // e.g. 1, 2, 3
    ExitCommitmentDuration time.Duration // e.g. 5*365*24*time.Hour; after user triggers exit, they must wait this before claim; no bonus after
    BonusAPY               sdk.Dec       // e.g. 0.10 for 10% per year (fixed APY on locked amount)
    MinLockAmount          math.Int      // minimum amount (in bond denom) required when creating a new position in this tier; enforced on MsgLockTier
}

// Params
type Params struct {
    Tiers []TierDefinition
    BonusDenoms []string  // denom(s) for bonus payouts (e.g. bond denom)
}
```

- **Minimum lock:** Each tier’s `MinLockAmount` must be ≥ 0 when params are set or updated; implementations may require it to be strictly positive for the tier to accept new locks. It is enforced only on **MsgLockTier** (new positions); **MsgAddToTierPosition** does not require the added amount to meet any minimum.
- **Fixed APY:** Bonus is not a multiplier on base rewards. It is an annual rate on the locked (or delegated) amount: `accrued_bonus = amount_locked × BonusAPY × (time_elapsed / 1 year)`, paid from the tier pool in `BonusDenoms`. Accrual can be computed per block or on each withdraw/claim using `LastAccrualTime` (or height) stored on the position.

### 4.2 Tier positions (state)

Tier locks are stored as **state records** in the module. Each record represents one lock and its delegation state. A tier position **cannot be broken down**: when delegated, the **full** `amount_locked` is delegated to a single validator (all-or-nothing).

```go
// TierPosition: one per lock. Key = position_id (unique).
type TierPosition struct {
    PositionId       uint64    // unique ID (e.g. incrementing counter)
    Owner            string   // sdk.AccAddress
    TierId           uint32
    AmountLocked     math.Int // tokens locked (bond denom); when delegated, this full amount is delegated
    CreatedAtHeight  int64
    CreatedAtTime    time.Time
    // Exit commitment: once user triggers exit, they must wait until ExitUnlockTime to claim; no bonus after that.
    ExitTriggeredAt  time.Time // zero = not triggered; when set, position is "exiting"
    ExitUnlockTime   time.Time // ExitTriggeredAt + ExitCommitmentDuration; when user can claim and bonus stops
    // Delegation state: position is either not delegated (Validator empty) or fully delegated to one validator.
    Validator        string   // validator operator address if delegated; empty if not
    DelegatedShares  sdk.Dec  // validator delegation shares for this position (see 4.2.1)
    LastBonusAccrual time.Time // for fixed APY: last time bonus was accrued
}

// Store layout:
// - PositionByID:       position_id -> TierPosition
// - PositionsByOwner:  owner || position_id -> TierPosition (for listing by owner)
// - NextPositionId:     next uint64
```

- **Lock into tier:** User sends `MsgLockTier(tier_id, amount)`. The amount must be **≥** the tier’s **MinLockAmount**. Tokens are transferred to the tier module account; module creates a `TierPosition` with `amount_locked`, no exit triggered, no validator (not yet delegated). Multiple positions per owner allowed. User can trigger exit at any time.
- **Add to position:** Owner can send `MsgAddToTierPosition(position_id, amount)` to add tokens to an **existing** position. Allowed only while exit has **not** been triggered (`ExitTriggeredAt` is zero). See §4.6 for what is updated on add.
- **Delegation:** When the user calls `MsgTierDelegate(position_id, validator)`, the module delegates the **full** `amount_locked` to that validator (position cannot be split). The staking module returns **shares** for that delegation; the tier module stores them in `DelegatedShares` (see §4.2.1). **Only once delegated** do base and bonus rewards accrue; bonus APY accrues on `amount_locked` from the time of delegation (see §4.5).

#### 4.2.1 Delegated shares calculation

A tier position is always delegated as a **whole**: the full `amount_locked` is delegated in a single `staking.Delegate` call. The **delegated shares** are the shares returned by the staking module for that delegation.

- **On delegate:** The tier module calls `stakingKeeper.Delegate(ctx, tierModuleAccount, validator, position.AmountLocked)` (bond denom). The staking module converts the token amount to validator delegation shares using the validator’s current exchange rate (tokens per share) and returns the `newShares` created. The tier module sets `position.Validator = validator`, `position.DelegatedShares = newShares`, and `position.LastBonusAccrual = block_time` so that bonus (and base) rewards accrue only from delegation time.
- **On undelegate:** Allowed **only when the position has triggered exit** (`ExitTriggeredAt != 0`). The tier module calls `stakingKeeper.Undelegate(ctx, tierModuleAccount, position.Validator, position.DelegatedShares)` — the **full** shares for this position. No partial undelegate; the position’s entire delegation is unbonded. While the user has not triggered exit, **MsgTierUndelegate** is rejected so that tier lockers stay delegated and keep providing security.
- **On redelegate:** The tier module calls `stakingKeeper.BeginRedelegate(ctx, tierModuleAccount, position.Validator, dstValidator, position.DelegatedShares)`. The full shares are moved to the destination validator; when the redelegation completes, the position’s `Validator` is updated to the destination and `DelegatedShares` is set to the **new shares** on the destination validator (staking returns the shares issued on the destination; that value is stored).

Because each position is a single delegation of a fixed amount, there is a 1:1 mapping between a tier position and one delegation entry in staking (for that validator). Base rewards for that delegation are attributed entirely to that position’s owner when they call `MsgWithdrawTierRewards`.

### 4.3 Withdraw from tier (two phases)

The user can **trigger exit at any time**. Withdraw is a **two-phase** process:

**Phase 1 — Trigger exit**

- **Message:** `MsgTriggerExitFromTier(position_id)`. Auth: signer must be `TierPosition.Owner`. Reject if already exiting (`ExitTriggeredAt != 0`). Set `ExitTriggeredAt = block_time`, `ExitUnlockTime = ExitTriggeredAt + tiers[tier_id].ExitCommitmentDuration`. Position is now in **exiting** state.
- User must wait until `block_time >= ExitUnlockTime` before they can claim. During the exit commitment period, bonus can still accrue; **once exit commitment has elapsed** (`block_time >= ExitUnlockTime`), **no more bonus** is paid for this position.

**Phase 2 — Claim tokens**

- Allowed only when `block_time >= position.ExitUnlockTime` (exit commitment elapsed). Reject otherwise.
- **Message:** `MsgWithdrawFromTier(position_id)` (or `MsgClaimFromTier(position_id)`). Auth: signer must be `TierPosition.Owner`. If position is delegated, require no active delegation and no unbonding entries (user must have undelegated and waited for unbonding). Transfer `amount_locked` from tier module to owner, delete position, emit event.

So: **lock → user may trigger exit anytime → wait X years (exit commitment, per tier) → no more bonus → claim tokens**. Optional `MsgClaimExpiredTier(position_id)` for positions that have reached `ExitUnlockTime` (same as claim above).

### 4.4 Tier rewards pool

- **Option 1:** Module account `tieredrewards` (or `tier_reward_pool`). External logic sends coins to this account (e.g. via `MsgFundTierPool` with authority, or from another module).  
- **Option 2:** External keeper interface (like distribution’s `ExternalCommunityPoolKeeper`): the tier module calls `WithdrawFromTierPool(ctx, amount)` and the external keeper moves coins to the delegator.  

Recommendation: **Module account** for simplicity; an optional **external funder** can be allowed via authority-restricted `MsgFundTierPool`.

### 4.5 How tier rewards are tracked and calculated

Tier rewards consist of **base rewards** (from staking/distribution) and **bonus rewards** (fixed APY from the tier pool). They are tracked and calculated as follows.

#### Base rewards (staking)

- **Where they are tracked:** In `x/distribution`, not in the tier module. The distribution module keeps:
  - **Validator historical rewards:** cumulative reward ratio per validator per period (updated when a period is closed and when rewards are allocated).
  - **Delegator starting info:** per (delegator, validator) the starting period and the delegator's cumulative ratio cursor, so that delegator reward = shares × (current_ratio − starting_ratio).
- **Tier's use:** The **tier module account** is the delegator. Each tier position is one delegation (full amount to one validator), so there is a 1:1 mapping: one position ⇒ one (tier_module, validator) delegation in staking. When the owner calls `MsgWithdrawTierRewards(position_id)`, the tier module calls `distribution.WithdrawDelegationRewards(ctx, tier_module_account, position.Validator)`. Distribution computes the accrued base rewards for that delegation from its stored ratios and starting info, sends the coins to the tier module's withdraw address, and updates the delegator's starting info. The tier module then sends that full amount to the position owner (no splitting, because one position = one delegation).
- **No per-position base state in tier:** The tier module does not duplicate distribution's reward state; it only stores which validator (and shares) each position has, so it knows which delegation to withdraw for when the user withdraws rewards.

#### Bonus rewards (fixed APY)

- **Where they are tracked:** In the tier module, **per position**:
  - **`LastBonusAccrual`** (time): the last time up to which bonus has been accrued (and paid or committed). Bonus for the interval from `LastBonusAccrual` to the accrual end time is computed and paid when the user withdraws; then `LastBonusAccrual` is set to that end time.
- **How they are calculated:** When paying tier rewards (e.g. in `MsgWithdrawTierRewards`), **the position must be delegated** (`Validator` set); otherwise skip bonus (and base is already zero when not delegated). Steps:
  1. **Accrual end time:** `accrual_end = block_time`. If the position is exiting (`ExitTriggeredAt` set) and `block_time > ExitUnlockTime`, set `accrual_end = ExitUnlockTime` so no bonus is paid for time after the exit commitment has elapsed.
  2. **Time elapsed:** `time_elapsed = accrual_end − position.LastBonusAccrual` (duration in seconds or same unit as the tier's APY period). If the position was undelegated for some period, `LastBonusAccrual` is set when they delegate again, so only delegated time is counted.
  3. **Accrued bonus:**  
     `accrued_bonus = position.AmountLocked × tier.BonusAPY × (time_elapsed / 1 year)`  
     in the tier's bonus denom(s). The result is capped to the tier pool's available balance so the module never sends more than it holds.
  4. **Pay and update:** Send `accrued_bonus` from the tier rewards pool to the position owner. Set `position.LastBonusAccrual = accrual_end`.
- **When accrual starts:** Set when the position is **first delegated** (e.g. `LastBonusAccrual = block_time` in `MsgTierDelegate`). **No bonus accrues while the position is not delegated** — this forces lockers to delegate to a validator to earn rewards and thus provide security to the network. If the position was previously undelegated and is delegated again, set `LastBonusAccrual = block_time` when delegating so bonus accrues only from that time.
- **When accrual stops:** (1) When the position is **not delegated** (`Validator` empty), no bonus accrues. (2) For positions that have triggered exit, bonus stops at `ExitUnlockTime` (no bonus for time after the exit commitment has elapsed). The calculation above enforces this by capping `accrual_end` at `ExitUnlockTime`. When paying bonus in `MsgWithdrawTierRewards`, the position must be delegated; if not delegated, no bonus is paid for the undelegated period.

#### Summary

| Reward type | Tracked in | Tier module state used | When calculated |
|------------|------------|------------------------|------------------|
| Base       | `x/distribution` (validator historical rewards, delegator starting info) | `position.Validator` (and 1:1 delegation) | On `MsgWithdrawTierRewards`: **only if position is delegated**; call distribution withdraw for (tier_module, position.Validator); forward full amount to owner. No base rewards when not delegated. |
| Bonus      | Tier module: `position.LastBonusAccrual` | `LastBonusAccrual`, `AmountLocked`, tier `BonusAPY`, `ExitUnlockTime` if exiting | On withdraw: **only if position is delegated**; `accrued = AmountLocked × BonusAPY × (accrual_end − LastBonusAccrual) / 1 year`; cap to pool; pay to owner; set `LastBonusAccrual = accrual_end`. No bonus accrues or is paid when not delegated. |

### 4.6 When the tier position must be updated

The tier position is a state record that must stay consistent with staking and with user actions. The following cases require **updating** the position (or deleting it).

| Trigger | What to update | Notes |
|--------|----------------|--------|
| **MsgTierDelegate** | `Validator`, `DelegatedShares`, `LastBonusAccrual` | Set to the chosen validator and the shares returned by staking. Set `LastBonusAccrual = block_time` so bonus accrual starts from delegation time (rewards only when delegated). |
| **MsgAddToTierPosition** | See “Add to position” below | Allowed only when exit not triggered. Updates listed in next subsection. |
| **Unbonding completes** (after MsgTierUndelegate) | `Validator` → empty, `DelegatedShares` → 0 | When the tier module’s unbonding for this position’s shares completes, the position is no longer delegated. Clear delegation state so the position is “undelegated”; **no rewards (base or bonus) accrue until the owner delegates again**. Requires tracking which unbonding entry belongs to which position (e.g. via staking hook or EndBlocker that matches completion to position). |
| **Redelegation completes** (after MsgTierRedelegate) | `Validator` → destination validator, `DelegatedShares` → new shares on destination | Staking returns the shares issued on the destination; store them so future undelegate/withdraw uses the correct shares. |
| **MsgTriggerExitFromTier** | `ExitTriggeredAt`, `ExitUnlockTime` | Position enters “exiting” state; bonus stops at `ExitUnlockTime`. |
| **MsgWithdrawTierRewards** | `LastBonusAccrual` → accrual_end | So bonus is not double-counted for the same period. |
| **MsgTransferTierPosition** (optional) | `Owner` | Transfer ownership; no change to delegation or exit state. |
| **Slashing** | `AmountLocked` (and possibly `DelegatedShares`) | When the validator or the tier module’s delegation is slashed, the token value of the delegation drops. Update the position so **AmountLocked** reflects the **current** value of the delegation (e.g. tokens from `staking.TokensFromShares(position.Validator, position.DelegatedShares)` or equivalent). Otherwise (1) bonus APY would accrue on the pre-slash amount, and (2) on claim the module would owe `AmountLocked` but only hold the slashed amount. Implementation: use a staking hook (e.g. `AfterValidatorSlashed` / `BeforeValidatorSlashed`) or periodic reconciliation to detect slashing and update each affected position’s `AmountLocked` (and `DelegatedShares` if the chain reduces shares on slash). |
| **MsgWithdrawFromTier** (claim) | Position **deleted** | Not an update; the position is removed after tokens are sent to the owner. |
| **Validator leaves active set** (jailed, unbonding, removed) | No mandatory update | The delegation still exists in staking (the validator may be jailed, unbonding, or unbonded). The position’s `Validator` and `DelegatedShares` remain valid; the user can call **MsgTierRedelegate** to move to another validator or **MsgTierUndelegate** to unbond. No new block rewards are earned while the validator is inactive. **No automatic position update** is required. Optional: if the chain implements auto-undelegate when a validator is removed (e.g. for safety), then when that unbonding completes the position is updated as in “Unbonding completes” above. |

**Summary:** Delegation state (`Validator`, `DelegatedShares`) is updated on delegate, on unbonding completion (only after exit-triggered undelegate), and on redelegation completion. Exit state is updated on trigger exit. **Undelegate is only allowed after trigger exit** so tier lockers stay delegated until they leave. Reward state (`LastBonusAccrual`) is updated on withdraw rewards. **Add to position** updates `AmountLocked` (and delegation state if already delegated). **Slashing** requires updating `AmountLocked` (and possibly `DelegatedShares`) so the position reflects the current value of the delegation and bonus/payout are correct. When a **validator leaves the set**, the user should redelegate (undelegate is only allowed when exiting).

#### Add to position: what is updated (MsgAddToTierPosition)

When the owner adds tokens to an existing position (`position_id`), the following are updated. **Precondition:** `ExitTriggeredAt` is zero (exit has not been triggered).

| Field / action | Update |
|----------------|--------|
| **AmountLocked** | Increase by the added `amount`: `position.AmountLocked += amount`. Tokens are transferred from the owner to the tier module before this update. |
| **Validator** | Unchanged (position remains delegated to the same validator or remains undelegated). |
| **DelegatedShares** | **If position is delegated** (`Validator != ""`): call `staking.Delegate(ctx, tier_module_account, position.Validator, amount)` to delegate the **new** tokens to the same validator. Staking returns `newShares`. Set `position.DelegatedShares += newShares`. **If position is not delegated:** no change. |
| **LastBonusAccrual** | **Option A (simple):** Leave unchanged. Bonus will accrue on the full (increased) `AmountLocked` for the period since `LastBonusAccrual`; the newly added amount is effectively credited with bonus from the last accrual time (slightly favorable to the user). **Option B (fair):** Before adding, settle bonus to now: compute and pay accrued bonus on the **current** `AmountLocked` from `LastBonusAccrual` to `block_time`, then set `LastBonusAccrual = block_time`. Then add the amount. The new tokens accrue bonus only from add-time onward. |
| **CreatedAtHeight**, **CreatedAtTime** | Unchanged (position creation time is preserved). |
| **ExitTriggeredAt**, **ExitUnlockTime** | Unchanged (must be zero / unset for add to be allowed). |
| **Owner**, **TierId**, **PositionId** | Unchanged. |

---

## 5. Message Flows

### 5.1 Lock into tier

```
User -> MsgLockTier(tier_id, amount)
  -> Validate tier_id exists; amount > 0; denom = bond denom; amount >= tiers[tier_id].MinLockAmount
  -> Bank.SendCoinsFromAccountToModule(owner, tieredrewards.ModuleName, amount)
  -> position_id = NextPositionId; NextPositionId++
  -> Set TierPosition(position_id, owner, tier_id, amount_locked=amount, CreatedAtHeight, CreatedAtTime, ExitTriggeredAt=0, ExitUnlockTime=0, Validator="", DelegatedShares=0, LastBonusAccrual=now)
  -> Index by owner (PositionsByOwner)
  -> Emit event (position_id, owner, tier_id, amount)
```

Tokens are held by the tier module. Position is not yet delegated; user can call `MsgTierDelegate` next.

### 5.2 Add to existing position

```
User -> MsgAddToTierPosition(position_id, amount)
  -> Auth: signer == TierPosition.Owner
  -> Load TierPosition; require position exists; require ExitTriggeredAt is zero (exit not triggered)
  -> Validate amount > 0; denom = bond denom
  -> Bank.SendCoinsFromAccountToModule(owner, tieredrewards.ModuleName, amount)
  -> position.AmountLocked += amount
  -> If position is delegated (Validator != ""):
       newShares = Staking.Delegate(ctx, tier_module_account, position.Validator, amount)
       position.DelegatedShares += newShares
  -> Optional (Option B): settle bonus to now (pay accrued on current amount, set LastBonusAccrual = block_time) before adding
  -> Save TierPosition
  -> Emit event (position_id, owner, amount_added, new_total)
```

Only the owner can add, and only while the position has not triggered exit. See §4.6 “Add to position: what is updated” for the full list of fields updated.

### 5.3 Tier delegate / undelegate / redelegate (internal only)

Tier-locked tokens can only be staked **within** the tier mechanism. The **tier module account** is the delegator in `x/staking`. A tier position **cannot be broken down**: delegation is always the **full** amount of the position. **Tier lockers cannot undelegate until they have triggered exit** — this keeps them delegated and providing security to the network until they commit to leaving.

**MsgTierDelegate(position_id, validator)**

```
  -> Auth: signer == TierPosition.Owner
  -> Load TierPosition; require position not already delegated (Validator == "")
  -> Staking.Delegate(ctx, tier_module_account, validator, position.AmountLocked)
  -> Staking returns newShares (shares issued for this delegation)
  -> Update position: Validator = validator, DelegatedShares = newShares; LastBonusAccrual = block_time (rewards start only when delegated)
  -> Emit event (position_id, owner, validator)
```

**MsgTierUndelegate(position_id)**

```
  -> Auth: signer == TierPosition.Owner
  -> Load TierPosition; require position is delegated (Validator != ""); require position has triggered exit (ExitTriggeredAt != 0) — reject otherwise (no voluntary undelegation while in tier)
  -> Staking.Undelegate(ctx, tier_module_account, position.Validator, position.DelegatedShares)
  -> Unbonding is created. When unbonding completes, clear position.Validator and position.DelegatedShares (or track unbonding and update position state on completion).
  -> Emit event (position_id, owner)
```

**MsgTierRedelegate(position_id, dst_validator)**

```
  -> Auth: signer == TierPosition.Owner
  -> Load TierPosition; require position is delegated (Validator != "")
  -> Staking.BeginRedelegate(ctx, tier_module_account, position.Validator, dst_validator, position.DelegatedShares)
  -> When redelegation completes: set position.Validator = dst_validator, position.DelegatedShares = newShares (shares issued on destination validator)
  -> Emit event (position_id, owner, dst_validator)
```

These tokens **cannot** be used outside the tier module (no external LST); they are internal to this mechanism.

### 5.4 Trigger exit from tier

```
User -> MsgTriggerExitFromTier(position_id)
  -> Auth: signer == TierPosition.Owner
  -> Load TierPosition; reject if already exiting (ExitTriggeredAt != 0)
  -> ExitTriggeredAt = block_time; ExitUnlockTime = ExitTriggeredAt + tiers[tier_id].ExitCommitmentDuration
  -> Update TierPosition
  -> Emit event (position_id, owner, ExitUnlockTime)
```

User stays in tier (earning base + bonus) until they trigger exit. After triggering, they must wait until `ExitUnlockTime` to claim; once that time has passed, no more bonus.

### 5.5 Withdraw from tier (claim after exit commitment)

```
User -> MsgWithdrawFromTier(position_id)
  -> Auth: signer == TierPosition.Owner
  -> Load TierPosition; require block_time >= position.ExitUnlockTime (exit commitment elapsed). If ExitTriggeredAt is zero, reject (must trigger exit first).
  -> If delegated: require no active delegation and no unbonding entries (user must have undelegated and unbonding completed)
  -> Bank.SendCoinsFromModuleToAccount(tier_module, owner, position.amount_locked)
  -> Delete TierPosition(position_id)
  -> Emit event (position_id, owner)
```

No bonus is paid after `ExitUnlockTime`; this message only transfers tokens and burns the position.

### 5.6 Withdraw tier rewards (base + fixed APY bonus)

See **§4.5** for how base and bonus rewards are tracked and calculated.

**Flow:** The module **withdraws the base reward first** (from `x/distribution` into the tier module), then **redistributes to the tier locker** (forwards base to the position owner and pays bonus from the tier pool to the owner).

```
User -> MsgWithdrawTierRewards(position_id)
  -> Auth: signer == TierPosition.Owner
  -> Load TierPosition; must be delegated (Validator set)

  --- Phase 1: Withdraw base reward ---
  -> distribution.WithdrawDelegationRewards(ctx, tier_module_account, position.Validator)
       (Distribution sends accrued base rewards to the tier module’s withdraw address.
        The tier module receives the full base reward for this delegation; one position = one delegation, so the full amount is attributed to this position.)

  --- Phase 2: Redistribute to the tier locker ---
  -> Base: Send the base reward (received in Phase 1) from the tier module to the position owner.
  -> Bonus (fixed APY):
       accrual_end = block_time; if exiting (ExitTriggeredAt set) and block_time > ExitUnlockTime, accrual_end = ExitUnlockTime (no bonus after)
       accrued = amount_locked × tier.BonusAPY × (accrual_end - position.LastBonusAccrual) / 1 year
       Cap to tier pool balance; send from tier rewards pool to position owner in BonusDenoms
  -> Update position.LastBonusAccrual = accrual_end
  -> Emit event (position_id, owner, base_amount, bonus_amount)
```

- **Order of operations:** Base is withdrawn from distribution and received by the module **before** any redistribution. The module then redistributes to the tier locker: first the base reward to the owner, then the bonus (from the tier pool) to the owner. This ensures base rewards are settled with distribution first; only then does the module pay out to the position owner.
- Base rewards: tier module is the delegator; its withdraw address can be set to itself so rewards arrive at the module, then the module attributes per position (by share of delegation) and sends to each owner when they call `WithdrawTierRewards`, or the module can use a single withdraw address per position (if the chain supports it). Simplest: one delegation per position so that a single withdraw gives one position’s base rewards to that owner.
- **Fixed APY** is accrued over time from `LastBonusAccrual`; if the position is exiting, accrual stops at `ExitUnlockTime` (no bonus after). Cap bonus to pool balance so users do not fail on insufficient pool.

#### Optimization: base reward withdrawal batching

In standard Cosmos SDK staking, there is **one delegation per (delegator, validator)**. The tier module is a single delegator, so it has **one delegation per validator** (all positions on that validator are aggregated into one delegation in staking). Base rewards are therefore accrued once per validator; attributing to a position requires computing that position’s **share** of the validator’s delegation (e.g. `position.DelegatedShares / total_tier_shares_on_validator`).

If every `MsgWithdrawTierRewards` called `distribution.WithdrawDelegationRewards(tier_module, position.Validator)`, the module would trigger one distribution withdrawal per user withdrawal. When many tier lockers are delegated to the same validator, that would cause redundant distribution calls and repeated withdrawal of the same logical reward pool (distribution updates delegator starting info after each withdraw, so later withdrawals in the same period would see reduced or zero new rewards unless we track a buffer).

**Recommended optimization:** withdraw base rewards **once per validator per block** (or per batch), then attribute from a **pending base rewards** buffer when tier lockers withdraw.

| Approach | Description |
|----------|--------------|
| **Lazy per-block** | On the **first** `MsgWithdrawTierRewards` in the block for a given validator `V`, call `distribution.WithdrawDelegationRewards(ctx, tier_module_account, V)`. Credit the received coins to a **pending base rewards** store keyed by validator (e.g. `PendingBaseRewards[V]`). For **every** `MsgWithdrawTierRewards` in that block (or in a later block, after a new withdrawal for `V` has run) for a position on `V`, compute the position’s share: `position.DelegatedShares / total_tier_delegated_shares_to_V`, send `share × PendingBaseRewards[V]` to the position owner, and deduct that amount from `PendingBaseRewards[V]`. So only **one** distribution withdrawal per validator per block, regardless of how many tier lockers on that validator withdraw. |
| **EndBlocker top-up** | Optionally, in **EndBlocker**, for each validator that has tier positions, call `WithdrawDelegationRewards` and add to `PendingBaseRewards[V]`. Then during the next block, user withdrawals only consume from the buffer and never call distribution. This reduces distribution calls from message handlers but may do one withdrawal per validator every block; use if the number of validators with tier positions is small or if EndBlocker cost is acceptable. |

**State:** Maintain `PendingBaseRewards[validator_addr] = sdk.Coins` (and optionally `LastWithdrawHeight[validator_addr]` if withdrawals are only allowed once per block per validator). Total tier shares per validator can be computed by iterating positions with that `Validator` or by maintaining a running total in state (updated on delegate / undelegate / redelegate).

**Attribution:** When a position on validator `V` withdraws, `position_base_share = position.DelegatedShares / total_delegated_shares_to_V`. Send `position_base_share × PendingBaseRewards[V]` to the owner (per denom), then subtract that from `PendingBaseRewards[V]`. This keeps rewards proportional to delegation share and avoids calling distribution on every tier locker withdrawal.

### 5.7 Fund tier pool (authority or external)

```
Authority / External module -> MsgFundTierPool(amount)
  -> Bank.SendCoinsFromAccountToModule(sender, tieredrewards.ModuleName, amount)
  -> Emit event
```

Optional: restrict sender to governance or a dedicated “rewards treasury” module.

---

## 6. Queries

- **Tier params:** `TierParams` – list tier definitions and bonus denoms.
- **Tier pool:** `TierPoolBalance` – balance of the tier module account (or external pool) for bonus payouts.
- **Position by ID:** `TierPosition(position_id)` – return the full position record (owner, tier_id, amount_locked, exit_triggered_at, exit_unlock_time, validator, delegated_shares, created_at, etc.) by position ID.
- **Positions by owner:** `TierPositionsByOwner(owner, pagination)` – list all tier positions owned by an address (for wallets and UIs).
- **All positions:** `AllTierPositions(pagination)` – list all tier positions in the system (for explorers and analytics).
- **Estimate bonus:** `EstimateTierBonus(position_id)` – return estimated (base, bonus) for that position; useful for UX.
- **Voting power:** `TierVotingPower(owner)` – return voting power (sum of `AmountLocked` for delegated positions owned by `owner`); used by gov tally and by UIs to show tier locker’s governance power.

---

## 7. Integration with Staking and Distribution

- **Tier module as delegator:** The tier module account holds tier-locked tokens and is the **delegator** in `x/staking` for all tier delegations. The module calls `staking.Delegate`, `staking.Undelegate`, `staking.BeginRedelegate` with itself as delegator.
- **Base rewards:** When the tier module receives base rewards from `x/distribution`, the module attributes them to positions and sends to owners when they call `MsgWithdrawTierRewards`. Distribution withdraw address for the tier module can be the module account so rewards are received there and then forwarded. In standard SDK staking there is one delegation per (delegator, validator), so the tier module has **one delegation per validator** (aggregate of all positions on that validator); base rewards are attributed to positions by share. To avoid triggering a distribution withdrawal on every tier locker withdrawal, see **§5.6 Optimization: base reward withdrawal batching** (withdraw once per validator per block and attribute from a pending buffer).
- **Bonus:** Fixed APY is computed and paid from the tier pool on withdraw; no change to distribution logic.

---

## 8. Governance voting for tier lockers

Because the **tier module account** is the delegator in `x/staking`, staking-only governance tally attributes all tier-delegated voting power to the module address, not to the tier locker (position owner). Tier lockers would otherwise have no say in governance despite providing security via delegated tier positions. The following changes allow **tier lockers to vote with power proportional to their delegated tier-locked amount**.

### 8.1 Requirement

- Tier lockers vote using their **own address** (standard `MsgVote(proposal_id, option)` from `x/gov`).
- A voter’s **total voting power** = (staking voting power from their direct delegations) **+** (voting power from their **delegated** tier positions).
- Only positions that are **currently delegated** (`Validator` set) count; undelegated or unbonding positions do not count until they are delegated again.

### 8.2 Tier module: voting power API

The tier module must expose at least one of the following so that governance (or the app) can include tier power in the tally:

| Method | Description |
|--------|-------------|
| **GetVotingPowerForAddress(ctx, voterAddr) math.LegacyDec** | Returns the **voting power** (in bond denom units) for the given address: sum of `position.AmountLocked` over all tier positions where `position.Owner == voterAddr` and `position.Validator != ""` (position is delegated). Use the same unit as staking (e.g. bond denom amount) so it can be added to staking voting power. After slashing, `AmountLocked` reflects the current value, so this stays correct. |
| **TotalDelegatedVotingPower(ctx) math.LegacyDec** (optional) | Returns the sum of `AmountLocked` over all delegated positions. Used to include tier-delegated supply in the **total voting power** (quorum denominator) so that quorum and thresholds are consistent. If not implemented, quorum can remain staking-only (tier power only in the numerator). |

Implementations may use `AmountLocked` as the voting power per position (bond denom); if the chain uses validator share–based power for staking, the tier module can instead convert `DelegatedShares` to tokens via the validator’s exchange rate for consistency.

### 8.3 Gov (x/gov) changes

`x/gov` tallies votes using each voter’s **staking** delegations only (`sk.IterateDelegations(ctx, voter, ...)`). The tier module is the delegator for tier positions, so those delegations do not appear under the tier locker’s address. To let tier lockers vote:

- **Custom tally function:** Use the existing extension point **`WithCustomCalculateVoteResultsAndVotingPowerFn`** when constructing the gov keeper. The custom function:
  1. Reuses the default logic to compute each voter’s **staking** voting power and add it to the tally (iterate delegations from the voter, add to results and total).
  2. **For each voter,** calls the tier module’s **`GetVotingPowerForAddress(ctx, voterAddr)`** and adds that **tier voting power** to the same voter’s vote (same weight/options) and to `totalVotingPower`.
  3. Optionally, for **quorum**, the total bonded supply used in the quorum check can include tier-delegated supply: e.g. `totalBonded = sk.TotalBondedTokens(ctx) + tierKeeper.TotalDelegatedVotingPower(ctx)`. If the gov keeper does not have access to the tier keeper, the app can inject a custom tally that uses it and, if needed, a separate mechanism to adjust the quorum denominator (e.g. gov params or a hook that returns “effective total bonded” including tier).

- **Dependency:** Gov does not need to depend on the tier module at compile time; the **app** wires a `CalculateVoteResultsAndVotingPowerFn` that closes over the tier keeper (or an interface) and calls it when tallying. Gov’s `Tally()` continues to use `sk.TotalBondedTokens(ctx)` for quorum unless the app also provides a way to use an effective total that includes tier (e.g. custom tally that returns a different total, or a small change in `Tally` to accept an optional “total bonded” override).

### 8.4 Summary of changes

| Component | Change |
|-----------|--------|
| **Tier module** | Add `GetVotingPowerForAddress(ctx, voterAddr) math.LegacyDec` (sum of `AmountLocked` for delegated positions owned by `voterAddr`). Optionally add `TotalDelegatedVotingPower(ctx)` for quorum. |
| **App / Gov wiring** | Construct gov keeper with `WithCustomCalculateVoteResultsAndVotingPowerFn(fn)` where `fn` adds tier voting power per voter (via tier keeper). Optionally include tier total in quorum denominator. |
| **Voter experience** | Tier lockers vote with **MsgVote** as usual; no new message type. Their voting power automatically includes their delegated tier positions. |

### 8.5 Edge cases

- **Position not delegated:** No tier voting power for that position until the owner delegates via `MsgTierDelegate`.
- **Slashing:** `AmountLocked` is updated on slash (§4.6); tier voting power uses the updated amount.
- **Exit triggered, still delegated:** Position remains delegated until the owner calls `MsgTierUndelegate` and unbonding completes. Until then, it still counts for tier voting power.
- **Unbonding after undelegate:** Once the position’s delegation is cleared (`Validator` empty), it no longer counts. During unbonding, implementation may still have `Validator` set until completion; the ADR leaves that to the implementation (either exclude unbonding positions from `GetVotingPowerForAddress` or clear `Validator` only on unbonding completion).

---

## 9. Tier-Locked Tokens: Delegate / Undelegate / Redelegate Only via Tier

Tier-locked tokens **cannot** be delegated or redelegated using normal staking messages (undelegation only after trigger exit). They are **internal** to the tier mechanism:

- **MsgLockTier**, **MsgAddToTierPosition** (add to existing position while exit not triggered), **MsgTierDelegate(position_id, validator)**, **MsgTierRedelegate(position_id, dst_validator)** are the ways to lock or move tier-locked stake. **MsgTierUndelegate(position_id)** is **only allowed after the user has triggered exit** — tier lockers cannot voluntarily undelegate while in the tier. The tier module account is always the delegator; each position is delegated as a whole (full amount to one validator).
- When a user calls **MsgTierUndelegate** (only valid after trigger exit) or **MsgTierRedelegate**, staking runs as usual (unbonding/redelegation); distribution may run its hooks and pay base rewards to the tier module. The tier module attributes those rewards to the correct position(s). Design choice: require users to call **MsgWithdrawTierRewards** before **MsgTierUndelegate** so base + APY bonus are paid first.
- **Withdraw from tier:** User can trigger exit anytime with **MsgTriggerExitFromTier** (starts exit commitment, X years per tier); after exit commitment has elapsed, **MsgWithdrawFromTier** claims tokens (no more bonus). If delegated, user must have called **MsgTierUndelegate** (allowed only after trigger exit) and waited for unbonding before claim.

---

## 10. Edge Cases and Rules

| Case | Behavior |
|------|----------|
| Lock expired | No bonus; base only. Position can be claimed/burned via MsgClaimExpiredTier. |
| No lock | No bonus; base only. |
| Pool empty / insufficient | Base withdrawal still succeeds; bonus (fixed APY accrual) capped to available pool balance (or zero). |
| Slashing | The position must be updated when the delegation is slashed: set **AmountLocked** (and **DelegatedShares** if the chain changes shares) to the current value of the delegation (see §4.6). Base rewards and stake are reduced by distribution/staking; bonus APY then accrues on the updated amount. |
| Validator leaves set (jailed, unbonding, removed) | The delegation still exists; the position’s `Validator` and `DelegatedShares` stay valid. No new block rewards while the validator is inactive. User should **MsgTierRedelegate** to an active validator or **MsgTierUndelegate** to unbond. No mandatory position update (see §4.6). Optional: chain may auto-undelegate when a validator is removed; then unbonding-completion update applies. |
| Multiple denoms in base | Base rewards per denom; bonus can be restricted to bond denom or configurable. |
| Trigger exit / claim | User can trigger exit **anytime**. **MsgTriggerExitFromTier** starts exit commitment (wait X years per tier); then **MsgWithdrawFromTier** claims tokens. No bonus after exit commitment elapsed. If delegated, must undelegate and wait unbonding before claim. |
| Multiple positions per owner | Each position is an independent state record; bonus APY and base attribution per position. |
| Add to position when exit triggered | **Reject.** `MsgAddToTierPosition` is allowed only when `ExitTriggeredAt` is zero. Once the user has triggered exit, no further adds to that position. |
| Lock amount below tier minimum | **Reject.** `MsgLockTier(tier_id, amount)` requires `amount >= tiers[tier_id].MinLockAmount`. If amount is less than the tier’s minimum, the message fails validation. |
| Position not delegated | **No rewards.** Base and bonus rewards accrue **only when the position is delegated** to a validator. A locker who never delegates receives no rewards. Undelegation is only allowed after trigger exit, so while in the tier a position is either delegated (earning rewards) or in unbonding after exit; there is no voluntary "undelegated and idle" state. |

---

## 11. Security and Invariants

- **Authority:** Only designated authority (e.g. gov) can update tier params and fund the pool.  
- **Tier minimum lock:** For each tier, `MinLockAmount` is enforced on **MsgLockTier**: `amount >= tiers[tier_id].MinLockAmount`. Params must set `MinLockAmount` ≥ 0; implementations may require it to be positive for a tier to accept new locks.  
- **Rewards only when delegated:** Tier lockers receive **no** base or bonus rewards unless the position is delegated to a validator. This enforces that lockers contribute to network security to earn rewards.  
- **No undelegation until exit:** **MsgTierUndelegate** is only allowed when the position has triggered exit (`ExitTriggeredAt != 0`). Tier lockers cannot voluntarily undelegate while in the tier, so they keep providing security until they commit to leaving.  
- **No double bonus:** Bonus is fixed APY accrued over time; tracked per position (`LastBonusAccrual`); paid on `MsgWithdrawTierRewards` (and optionally on TierUndelegate/TierRedelegate).  
- **Pool balance:** Never send more than pool balance; cap bonus payout to available balance.  
- **Tier-only delegation:** Only tier module can delegate/undelegate/redelegate tier-locked tokens; staking messages from users do not affect tier positions.  
- **Withdraw address:** Base rewards attributed to positions and sent to owners; bonus paid to position owner.

---

## 12. Optional: Transfer tier position

If tier positions are **transferable**, add `MsgTransferTierPosition(sender, new_owner, position_id)`:

- Auth: `sender` must be current `TierPosition.Owner`.
- Update `TierPosition.Owner` to `new_owner`; keep tier_id and exit state unchanged.
- The new owner receives tier bonus and base rewards for this position; the previous owner no longer does.

---

## 13. Optional: Distribution Hook for Base Attribution

Because the **tier module** is the delegator in staking, base rewards are paid to the tier module. To attribute base rewards to individual positions (each position is one delegation, so attribution is per position), the chain can:

- Use a single withdraw address (the tier module account) and attribute base rewards internally by position when users call `MsgWithdrawTierRewards`, or  
- Extend `x/distribution` with a hook (e.g. `AfterDelegationRewardsWithdrawn`) so the tier module can attribute and forward base to position owners when distribution pays out.  

Bonus is **fixed APY** from the tier pool, not a multiplier on base; the hook would only affect how **base** rewards are attributed, not the bonus formula.

---

## 14. Module Layout

```
x/tieredrewards/
  keeper/
    keeper.go         # Keeper, params, pool balance, position store; GetVotingPowerForAddress (for gov tally)
    msg_server.go     # MsgLockTier, MsgAddToTierPosition, MsgTierDelegate, MsgTierUndelegate, MsgTierRedelegate, MsgTriggerExitFromTier, MsgWithdrawFromTier, MsgWithdrawTierRewards, MsgFundTierPool, MsgClaimExpiredTier, [MsgTransferTierPosition]
    query_server.go   # PositionByID, PositionsByOwner, AllTierPositions, params, pool
    position.go       # TierPosition CRUD, attribution
  types/
    keys.go
    params.go
    genesis.go
    messages.go
  module.go
  abci.go             # no-op or future pool refill logic
```

**Expected keepers:**

- `stakingKeeper`: `Delegate`, `Undelegate`, `BeginRedelegate` (tier module account as delegator).  
- `distributionKeeper`: withdraw base rewards to module; optional hook for attribution.  
- `bankKeeper`: send base + bonus to position owners; receive lock and pool funding.  
- `authKeeper`: module account, address codec.

---

## 15. Summary

- **Tier = lock duration + fixed bonus APY + minimum lock amount.** Each tier defines `MinLockAmount`; **MsgLockTier** requires `amount >= MinLockAmount` for that tier. **Rewards (base and bonus) are paid only when the position is delegated** to a validator, so lockers must provide security to the network to earn. **Undelegation is only allowed after the user has triggered exit** — tier lockers cannot voluntarily undelegate while in the tier. Tier-locked tokens are **internal** to the tier mechanism: **MsgTierDelegate**, **MsgTierRedelegate**, and **MsgTierUndelegate** (only after trigger exit) move stake; the tier module account is the delegator.  
- **Tier positions are state records:** each lock is a `TierPosition` (position_id, owner, tier_id, amount_locked, created_at, exit_triggered_at, exit_unlock_time, validator, delegated_shares, last_bonus_accrual). A position cannot be broken down; the full amount is delegated to one validator. The owner can **add** to an existing position via `MsgAddToTierPosition` as long as exit has not been triggered. Store: `PositionByID`, `PositionsByOwner`, `AllTierPositions`.  
- **Withdraw from tier:** User can trigger exit **at any time**. **MsgTriggerExitFromTier** starts the exit commitment (wait X years, per tier); once it has elapsed, **no more bonus** and **MsgWithdrawFromTier** claims tokens (after unbonding if delegated). Optional `MsgClaimExpiredTier` for positions past `ExitUnlockTime`.  
- **Bonus = fixed APY** on locked amount, accrued over time and paid from a **tier rewards pool** when user calls `MsgWithdrawTierRewards` (and optionally on TierUndelegate/TierRedelegate).  
- **Integration:** Tier module holds tokens and delegates via staking; base rewards received by module and attributed to positions; bonus from pool.  
- **Governance voting:** Tier lockers vote with their address; voting power includes their delegated tier positions via a custom gov tally and tier’s `GetVotingPowerForAddress` (§8).  
- **External pool:** Filled by governance or another module; tier module pays fixed-APY bonus from it.  
- **Safety:** Cap bonus to pool balance; only tier module can delegate/undelegate/redelegate tier-locked tokens; single authority for params and funding.
