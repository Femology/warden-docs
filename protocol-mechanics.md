# Protocol mechanics

Every number on this page is real — captured by actually calling `evaluate()` against
the live deployed contract (`CBFQ752LFNC57U4KWDAEKNU43PLBWJ7M2B4ZRYUMCWL62JHJNUYJVMB5`
on Testnet), not invented for illustration.

## The `evaluate()` lifecycle

`evaluate(wallet, recipient, amount)` is the one function called at the moment a
transfer is attempted. In order:

### 1. Load the policy

If the wallet has never called `set_policy`, this fails with `PolicyNotFound` —
    deliberately. An unconfigured wallet gets an explicit error, not a silent default
    decision.

### 2. Validate the amount

`amount` must be greater than zero, or the call fails with `InvalidAmount`.

### 3. Load or reset the velocity window

If 24 hours have passed since the window started, it resets: cumulative spend back
    to zero, transaction count back to zero, a new window start time. See
    [The velocity window](#the-velocity-window-and-its-known-limitation) below for the
    edge case this creates.

### 4. Decide, checking in this exact order

1. Is `new_recipient_requires_stepup` on, and is this recipient **not** in the
       trusted list? → `RequireStepUp(NewRecipient)`
    2. Is `amount` over `max_no_stepup`? → `RequireStepUp(AmountExceeded)`
    3. Would `cumulative_amount + amount` exceed `daily_velocity_cap`? →
       `RequireStepUp(VelocityExceeded)`
    4. Otherwise → `Allow`

    The first match wins. An amount that's both over the per-transaction limit *and*
    would exceed the velocity cap is reported as `AmountExceeded` — you'll see this
    below.

### 5. Update velocity — always, regardless of the decision

This runs whether the result was `Allow` or `RequireStepUp`. A transfer that
    triggered step-up and was then completed by the user still happened, and still
    counts toward the cap. If only `Allow`-ed transfers counted, someone could reset
    their effective velocity limit just by making every transfer trigger step-up.

### 6. Emit an event and return

`evaluation_allowed` or `stepup_required`, decoded and shown live in
    [`warden-monitor`](https://github.com/Femology/warden-monitor).


## Worked example: a real policy, four real transfers

This is an actual sequence run against the live contract. The wallet
(`GCZLMMKEOPOG5OB5QRLGH5ZKG7ACQNKX7KTT6UTXPFHUPS7FFSFFU5YM`) set this policy:

```json
{
  "maxNoStepUp": "150",
  "dailyVelocityCap": "500",
  "newRecipientRequiresStepUp": true
}
```

Then trusted one recipient (`GA7QYNF7SOWQ3GLR2BGMZEHXAVIRZA4KVWLTJJFC7MGXUA74P7UJVSGZ`).
Four transfers followed, to that same trusted recipient:

| # | Amount | Cumulative before | Decision | Why |
|---|---|---|---|---|
| 1 | 50.00 | 0.00001 | **Allow** | Trusted recipient, under the 150 limit, 50.00001 total is under the 500 cap |
| 2 | 200.00 | 50.00001 | **RequireStepUp** — `AmountExceeded` | 200 is over the 150 per-transfer limit (velocity isn't even checked — amount fails first) |
| 3 | 260.00 | 250.00001 | **RequireStepUp** — `AmountExceeded` | Same reason — 260 alone is over 150, regardless of velocity headroom |
| 4 | 10.00 | 510.00001 | **RequireStepUp** — `VelocityExceeded` | 10 is under the 150 limit on its own, but 510.00001 + 10 = 520.00001 is over the 500 cap |

Two things worth noticing in this real sequence:

- **Transfers 2 and 3 both triggered `AmountExceeded`, not `VelocityExceeded`** — even
  though by transfer 3 the wallet was already close to its daily cap. The amount check
  runs before the velocity check, so an over-limit single transfer is always reported
  as `AmountExceeded`, regardless of how much velocity headroom remains.
- **Every one of these four transfers added to cumulative spend**, including the three
  that triggered step-up. Cumulative went `0.00001 → 50.00001 → 250.00001 → 510.00001
  → 520.00001` — step-up transfers count exactly the same as allowed ones.

## The velocity window, and its known limitation

The window is **fixed, not sliding**. It tracks a `window_start` timestamp and resets
completely once `now - window_start >= 86400` seconds (24 hours) — it doesn't
continuously roll the last-24-hours total forward.

This creates one real edge case, stated plainly rather than hidden: a wallet could
spend right up to its daily cap in the last minute before a reset, then spend up to the
full cap again in the first minute after — briefly doubling its effective daily limit
across that boundary. For v1, this is a deliberate, accepted simplification, not an
oversight. A continuously sliding window is a reasonable improvement if this edge case
matters for a given deployment's risk tolerance — see
[warden-contract#3](https://github.com/Femology/warden-contract/issues/3).

## Reading a `Decision`

`evaluate()` returns one of two shapes:

```
Allow
RequireStepUp(AmountExceeded | NewRecipient | VelocityExceeded)
```

`warden-sdk` decodes this into a TypeScript discriminated union:

```ts
type Decision =
  | { type: 'Allow' }
  | { type: 'RequireStepUp'; reason: 'AmountExceeded' | 'NewRecipient' | 'VelocityExceeded' };
```

See the [Developer guide](developer-guide.md) for real, runnable code against this exact
deployed contract.
