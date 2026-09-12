# Contract reference

{% hint style="info" %}
Deployed on Stellar Testnet at
  [`CD5QU2E6LOKFAZFESIZSAA4IENH5SZHJVU4Y6532WNZSXPZDYRKEEVUW`](https://stellar.expert/explorer/testnet/contract/CD5QU2E6LOKFAZFESIZSAA4IENH5SZHJVU4Y6532WNZSXPZDYRKEEVUW).
  Source: [`warden-contract`](https://github.com/Femology/warden-contract).
{% endhint %}

{% hint style="warning" %}
This is the **Phases 14+15+16** master contract — dual velocity windows and trust decay
  (Phase 14), the flagged-address registry (Phase 15), and the guardian/recovery
  subsystem (Phase 16), all batch-deployed together as one instance. Two earlier
  instances are retired: the original pre-Phase-14 contract, and a Phase-14-only
  contract that came after it. Neither Phase 15 nor 16 broke `Policy`'s stored shape —
  they added independent storage keys — but this contract has no admin-upgrade
  function by design, so any new function can only ever ship as a new contract ID, not
  an in-place upgrade. See `warden-contract`'s README for the full migration history.
{% endhint %}

{% hint style="info" %}
This page covers the functions inherited from earlier phases in full. For Phase 15
  (flagged-address registry) and Phase 16 (guardian/recovery subsystem) specifically,
  see [`WARDEN-PROTOCOL.md`](https://github.com/Femology/warden-contract/blob/main/WARDEN-PROTOCOL.md)
  in `warden-contract` — the authoritative, language-neutral spec for every function,
  data type, event, and error this contract exposes, including the account state
  diagram and the governance process for changing any of it. This page is not
  duplicating that content function-by-function; treat the protocol doc as the source
  of truth for anything added since Phase 14.
{% endhint %}

## `initialize`

```
initialize(admin: Address, reference_asset: Address) -> Result<(), WardenError>
```

**Who can call it:** `admin` must sign (`admin.require_auth()`).
**What triggers it:** deploy-time only, once. Not part of the SDK's public surface —
it's a one-off deployment script, never something an integrating app calls.
**Fails with:** `AlreadyInitialized` if called a second time.

## `set_policy`

```
set_policy(
  wallet: Address,
  max_no_stepup: i128,
  daily_velocity_cap: i128,
  new_recipient_requires_stepup: bool,
  hourly_velocity_cap: i128,
  trust_decay_seconds: u64,
) -> Result<(), WardenError>
```

**Who can call it:** the wallet's own owner (`wallet.require_auth()`).
**What triggers it:** a wallet owner setting or changing their risk tolerance — the core
settings action.
**Behavior:** creates a policy if none exists (with an empty trusted-recipients map), or
updates the five fields in place if one does — `trusted_recipients` is left untouched by
this call, managed only by the two functions below.
**Fails with:** `InvalidPolicyParams` if `max_no_stepup < 0`, if
`daily_velocity_cap < max_no_stepup` (a velocity cap below the per-transfer threshold is
nonsensical, so it's rejected, not silently clamped), or if `hourly_velocity_cap < 0` or
`hourly_velocity_cap > daily_velocity_cap` (an hourly cap looser than the daily one would
never bind, making the hourly window meaningless).
**Emits:** `policy_set` — topics `("policy_set", wallet)`, data
`(max_no_stepup, daily_velocity_cap, hourly_velocity_cap, new_recipient_requires_stepup,
trust_decay_seconds)`.

## `add_trusted_recipient`

```
add_trusted_recipient(wallet: Address, recipient: Address) -> Result<(), WardenError>
```

**Who can call it:** the wallet's own owner.
**What triggers it:** the user saving a payee — the ordinary "add to trusted contacts"
action.
**Behavior:** stores `last_paid_at = now` for this recipient (see
[Protocol mechanics](protocol-mechanics.md#trust-decay) for what that's for). Every
successful `evaluate()` call to an already-trusted recipient refreshes this same
timestamp — it isn't only set once at add-time.
**Fails with:** `PolicyNotFound` if `set_policy` was never called first. `RecipientAlreadyTrusted`
if already present.
**Emits:** `recipient_trusted` — topics `("recipient_trusted", wallet)`, data `recipient`.

## `remove_trusted_recipient`

```
remove_trusted_recipient(wallet: Address, recipient: Address) -> Result<(), WardenError>
```

**Who can call it:** the wallet's own owner.
**What triggers it:** revoking a payee — a compromised address, a wrong entry, no
longer needed.
**Fails with:** `RecipientNotTrusted` if the recipient isn't in the list.
**Emits:** `recipient_untrusted` — topics `("recipient_untrusted", wallet)`, data
`recipient`.

## `evaluate`

```
evaluate(wallet: Address, recipient: Address, amount: i128) -> Result<Decision, WardenError>
```

**Who can call it:** the wallet's own owner.
**What triggers it:** the moment a transfer is attempted — this is the core function.
See [Protocol mechanics](protocol-mechanics.md) for the full decision lifecycle and real
worked numbers.
**Fails with:** `PolicyNotFound` if unconfigured. `InvalidAmount` if `amount <= 0`.
**Emits:** `evaluation_allowed` (data `(recipient, amount)`) on `Allow`, or
`stepup_required` (data `(recipient, amount, reason)`) on `RequireStepUp` — topics
`("eval_allowed", wallet)` / `("stepup_req", wallet)` respectively.

## `get_policy`

```
get_policy(wallet: Address) -> Result<Policy, WardenError>
```

**Who can call it:** anyone — public read, no auth. Policy data isn't secret; it's
config, and everything on a public ledger is inspectable regardless.
**Fails with:** `PolicyNotFound` if none is set.
**What triggers it:** the app's settings screen displaying current policy back to the
owner, or `warden-monitor`'s wallet drill-down page (read live, never cached).

## `get_velocity`

```
get_velocity(wallet: Address) -> Result<VelocityWindow, WardenError>
```

**Who can call it:** anyone — public read, no auth.
**Never errors on absence** — unlike `get_policy`, a wallet with no activity yet gets
back a zeroed window, since "no activity" is a normal read state, not a
missing-configuration error.
**What triggers it:** showing how close a wallet is running to its daily cap.

## Data types

```rust
pub struct Policy {
    pub owner: Address,
    pub max_no_stepup: i128,
    pub daily_velocity_cap: i128,
    pub hourly_velocity_cap: i128,
    pub new_recipient_requires_stepup: bool,
    // Address -> last_paid_at (unix seconds). A Map, not a Vec: this is
    // looked up by key on every evaluate() call -- get/contains_key/set/
    // remove is exactly Map's access pattern, and insertion order is never
    // used anywhere, so a Vec<(Address, u64)> would only add a linear scan
    // with no benefit.
    pub trusted_recipients: Map<Address, u64>,
    pub trust_decay_seconds: u64,
    pub updated_at: u64,
}

pub struct VelocityWindow {
    pub window_start: u64,
    pub cumulative_amount: i128,
    pub tx_count: u32,
}

pub enum Decision {
    Allow,
    RequireStepUp(StepUpReason),
}

pub enum StepUpReason {
    AmountExceeded,
    NewRecipient,
    VelocityExceeded,
    HourlyVelocityExceeded,
    FlaggedRecipient, // Phase 15 -- see WARDEN-PROTOCOL.md
}
```

{% hint style="warning" %}
`warden-sdk` decodes `Map<Address, u64>` as `Record<string, bigint>` (address ->
  last_paid_at) — a plain JS object, not an array. Verified against the live contract:
  stellar-sdk's own generic scval decoder returns a Soroban map with non-string keys
  (an `Address` is neither a plain string nor a symbol) as an array of `[key, value]`
  tuples, not a plain object, and `warden-sdk` converts that explicitly. If you're
  decoding the raw XDR yourself rather than going through `warden-sdk`, expect the
  tuple-array shape, not an object.
{% endhint %}

All amounts are `i128`. No monetary value anywhere in this contract, or anything built
on top of it, is ever represented as a float.

## Errors

```rust
pub enum WardenError {
    NotInitialized = 1,
    AlreadyInitialized = 2,
    PolicyNotFound = 3,
    InvalidAmount = 4,
    InvalidPolicyParams = 5,
    RecipientAlreadyTrusted = 6,
    RecipientNotTrusted = 7,
    // 8 through 20 are Phase 15 (flagged-address registry) and Phase 16
    // (guardian/recovery subsystem) -- see WARDEN-PROTOCOL.md's own error
    // table for the full list with meanings; not duplicated here.
}
```

`warden-sdk` maps these to a typed `WardenSdkError` by regex-matching the host's raw
failure text (`Error(Contract, #N)`) — verified against the real deployed contract, not
assumed.
