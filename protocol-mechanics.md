# Protocol mechanics

Every number on this page is real, captured by actually calling `evaluate()` against
the live deployed contract (`CD5QU2E6LOKFAZFESIZSAA4IENH5SZHJVU4Y6532WNZSXPZDYRKEEVUW`
on Testnet), not invented for illustration.

## The `evaluate()` lifecycle

`evaluate(wallet, recipient, amount)` is the one function called at the moment a
transfer is attempted. In order:

### 1. Load the policy

If the wallet has never called `set_policy`, this fails with `PolicyNotFound`,
    deliberately. An unconfigured wallet gets an explicit error, not a silent default
    decision.

### 2. Validate the amount

`amount` must be greater than zero, or the call fails with `InvalidAmount`.

### 3. Load or reset both velocity windows

Two independent windows, checked and reset the same way, on different periods:

- **Hourly**: 3,600 seconds.
- **Daily**: 86,400 seconds.

If a window's period has passed since its `window_start`, it resets: cumulative spend
    back to zero, transaction count back to zero, a new window start time. See
    [The velocity windows](#the-velocity-windows-and-their-known-limitation) below for
    the edge case this creates.

### 4. Check trust decay

A recipient in `trusted_recipients` only counts as trusted for step 5 below if
    `now - last_paid_at <= trust_decay_seconds`. Past that, they're still in the map
    (`add`/`remove_trusted_recipient` are the only calls that add or drop entries), but
    `evaluate()` treats them exactly like a recipient that was never trusted. See
    [Trust decay](#trust-decay) below.

### 5. Decide, checking in this exact order

{% hint style="info" %}
There's a step 0 ahead of everything below: is `recipient` present in the
  admin-managed flagged-address registry? → `RequireStepUp(FlaggedRecipient)`,
  regardless of amount, trust, or velocity headroom, and without consulting any of
  them. See
  [`WARDEN-PROTOCOL.md`](https://github.com/Femology/warden-contract/blob/main/WARDEN-PROTOCOL.md#the-evaluation-decision)
  for the exact, current, full order including this check.
{% endhint %}

1. Is `new_recipient_requires_stepup` on, and is this recipient **not** actively
       trusted (never trusted, or trusted but decayed)? → `RequireStepUp(NewRecipient)`
    2. Is `amount` over `max_no_stepup`? → `RequireStepUp(AmountExceeded)`
    3. Would `hourly_cumulative + amount` exceed `hourly_velocity_cap`? →
       `RequireStepUp(HourlyVelocityExceeded)`
    4. Would `daily_cumulative + amount` exceed `daily_velocity_cap`? →
       `RequireStepUp(VelocityExceeded)`
    5. Otherwise → `Allow`

    The first match wins. An amount that's both over the per-transaction limit *and*
    would exceed a velocity cap is reported as `AmountExceeded`; you'll see this
    below. Likewise, an amount that exceeds both windows at once is reported as
    `HourlyVelocityExceeded`, the more specific, more immediately actionable signal
    ("you're moving too fast right now" beats "you hit your day limit").

### 6. Update velocity, always, regardless of the decision

Both windows accumulate whether the result was `Allow` or `RequireStepUp`. A transfer
    that triggered step-up and was then completed by the user still happened, and
    still counts toward both caps. If only `Allow`-ed transfers counted, someone could
    reset their effective velocity limit just by making every transfer trigger
    step-up.

### 7. Refresh trust, if applicable

If the recipient is in `trusted_recipients` at all (decayed or not), their
    `last_paid_at` is refreshed to `now`. This does **not** touch the policy's own
    `updated_at`; that field means "the owner changed their configuration," not "a
    payment happened."

### 8. Emit an event and return

`evaluation_allowed` or `stepup_required`, decoded and shown live in
    [`warden-monitor`](https://github.com/Femology/warden-monitor).


## Worked example: a real policy, four real transfers

This is an actual sequence run against the live contract, against a freshly funded
Testnet wallet (`GCINF4I5LDWCW2ZLJKQEBOGFBACQNUETMPN6WYIEDPSFPTOTWLPCCPRC`) so the
velocity windows and trust state start from genuinely zero. It set this policy:

```json
{
  "maxNoStepUp": "150",
  "dailyVelocityCap": "500",
  "hourlyVelocityCap": "200",
  "newRecipientRequiresStepUp": true,
  "trustDecaySeconds": 2592000
}
```

Then trusted one recipient (`GA7QYNF7SOWQ3GLR2BGMZEHXAVIRZA4KVWLTJJFC7MGXUA74P7UJVSGZ`).
Four transfers followed, to that same trusted recipient, all inside the same hourly
window:

| # | Amount | Decision | Why |
|---|---|---|---|
| 1 | 50.00 | **Allow** | Trusted recipient, under the 150 limit, 50 total is under both the 200 hourly and 500 daily caps |
| 2 | 200.00 | **RequireStepUp**, `AmountExceeded` | 200 is over the 150 per-transfer limit (neither velocity window is even checked; amount fails first) |
| 3 | 100.00 | **RequireStepUp**, `HourlyVelocityExceeded` | 100 alone is under 150, but hourly cumulative is already 50 + 200 = 250 from transfers 1-2 (both counted, even the step-up one); 250 + 100 = 350 is over the 200 hourly cap |
| 4 | 100.00 | **RequireStepUp**, `HourlyVelocityExceeded` | Same reason; hourly cumulative is now 350, still well over the 200 cap |

Final `getVelocity` read:

```json
{ "windowStart": "1789205262", "cumulativeAmount": "450", "txCount": 4 }
```

Three things worth noticing in this real sequence:

- **Transfer 2 triggered `AmountExceeded`, not a velocity reason**, even though it was
  also large enough to blow through the hourly cap on its own. The amount check runs
  before either velocity check, so an over-limit single transfer is always reported as
  `AmountExceeded`, regardless of velocity headroom.
- **The hourly cap, not the daily one, is what actually bound here.** Cumulative spend
  (450) never got close to the 500 daily cap, but easily cleared the tighter 200 hourly
  one, exactly the scenario a burst-detection window exists for.
- **Every one of these four transfers added to cumulative spend**, including the three
  that triggered step-up. Cumulative went `0 → 50 → 250 → 350 → 450`; step-up
  transfers count exactly the same as allowed ones.

## Trust decay

`trust_decay_seconds` (set per-wallet in `set_policy`) controls how long a trusted
recipient stays trusted *without a payment*. `add_trusted_recipient` sets
`last_paid_at` to the current ledger time; every successful `evaluate()` call to that
recipient afterward refreshes it again, whatever the decision was.

If `now - last_paid_at` exceeds `trust_decay_seconds`, the recipient is still present
in `trusted_recipients` (`remove_trusted_recipient` is the only thing that drops an
entry), but `evaluate()` treats them as if `new_recipient_requires_stepup` applied to
them for the first time. One more payment refreshes `last_paid_at` and they're back to
being actively trusted.

The tradeoff: a recipient you pay often stays frictionless indefinitely, but one you
paid once two years ago and never again goes back to requiring confirmation; "trusted
because you dealt with them recently" is a meaningfully different, safer claim than
"trusted because you dealt with them once, ever."

## The velocity windows, and their known limitation

Both windows are **fixed, not sliding**. Each tracks its own `window_start` timestamp
and resets completely once its period has elapsed (3,600s for hourly, 86,400s for
daily); neither continuously rolls its trailing total forward.

This creates one known edge case: a wallet could spend right up to a cap in the last
moment before that window resets, then spend up to the full cap again right after,
briefly doubling its effective limit across that boundary. This applies independently
to both windows. It's an accepted simplification for now; a continuously sliding
window is a reasonable improvement if it matters for a given deployment's risk
tolerance. See [warden-contract#3](https://github.com/Femology/warden-contract/issues/3).

## Reading a `Decision`

`evaluate()` returns one of two shapes:

```
Allow
RequireStepUp(AmountExceeded | NewRecipient | VelocityExceeded | HourlyVelocityExceeded)
```

`warden-sdk` decodes this into a TypeScript discriminated union:

```ts
type Decision =
  | { type: 'Allow' }
  | {
      type: 'RequireStepUp';
      reason: 'AmountExceeded' | 'NewRecipient' | 'VelocityExceeded' | 'HourlyVelocityExceeded';
    };
```

See the [Developer guide](developer-guide.md) for real, runnable code against this exact
deployed contract.
