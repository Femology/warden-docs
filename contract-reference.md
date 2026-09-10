# Contract reference

{% hint style="info" %}
Deployed on Stellar Testnet at
  [`CBFQ752LFNC57U4KWDAEKNU43PLBWJ7M2B4ZRYUMCWL62JHJNUYJVMB5`](https://stellar.expert/explorer/testnet/contract/CBFQ752LFNC57U4KWDAEKNU43PLBWJ7M2B4ZRYUMCWL62JHJNUYJVMB5).
  Source: [`warden-contract`](https://github.com/Femology/warden-contract).
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
) -> Result<(), WardenError>
```

**Who can call it:** the wallet's own owner (`wallet.require_auth()`).
**What triggers it:** a wallet owner setting or changing their risk tolerance — the core
settings action.
**Behavior:** creates a policy if none exists (with an empty trusted-recipients list),
or updates the three fields in place if one does — `trusted_recipients` is left
untouched by this call, managed only by the two functions below.
**Fails with:** `InvalidPolicyParams` if `max_no_stepup < 0`, or if
`daily_velocity_cap < max_no_stepup` (a velocity cap below the per-transfer threshold
is nonsensical, so it's rejected, not silently clamped).
**Emits:** `policy_set` — topics `("policy_set", wallet)`, data
`(max_no_stepup, daily_velocity_cap, new_recipient_requires_stepup)`.

## `add_trusted_recipient`

```
add_trusted_recipient(wallet: Address, recipient: Address) -> Result<(), WardenError>
```

**Who can call it:** the wallet's own owner.
**What triggers it:** the user saving a payee — the ordinary "add to trusted contacts"
action.
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
    pub new_recipient_requires_stepup: bool,
    pub trusted_recipients: Vec<Address>,
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
}
```

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
}
```

`warden-sdk` maps these to a typed `WardenSdkError` by regex-matching the host's raw
failure text (`Error(Contract, #N)`) — verified against the real deployed contract, not
assumed.
