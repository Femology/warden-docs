# Developer guide

{% hint style="warning" %}
Every code sample on this page was run against the real deployed contract
  (`CD5QU2E6LOKFAZFESIZSAA4IENH5SZHJVU4Y6532WNZSXPZDYRKEEVUW` on Testnet) while writing
  this guide. The output shown is real, not invented.
{% endhint %}

## Install

```bash
npm install warden-sdk@github:Femology/warden-sdk#v0.4.0
```

`warden-sdk` isn't published to npm yet (tracked in
[warden-sdk#2](https://github.com/Femology/warden-sdk/issues/2)) — install it as a git
dependency pinned to a tag, as shown above. ESM only, Node ≥18.

{% hint style="info" %}
Pin `v0.4.0` or later. `v0.1.2` has a real bug where every `submit*` call throws
  `"The transaction has not yet been signed"` against a live network, regardless of
  whether you signed correctly (see
  [warden-sdk#5](https://github.com/Femology/warden-sdk/pull/5)). `v0.2.0` added
  support for this contract's Phase 14 fields but had two more real bugs, both found by
  actually running the examples below against the live network rather than trusting the
  mocked unit tests: `trustedRecipients` decoded with the wrong keys entirely (a numeric
  index instead of the actual address), and `getPolicy` crashed instead of returning
  `null` for a wallet with no policy set. Fixed in `v0.2.1`
  ([warden-sdk#7](https://github.com/Femology/warden-sdk/pull/7)). `v0.3.0` added
  Phase 15/16 support (flagged addresses, guardian recovery) and fixed a bug affecting
  every write call, old and new: a doomed call used to produce signable XDR anyway,
  failing only at submission with an opaque `tx_malformed` instead of the real
  `WardenError`
  ([warden-sdk#8](https://github.com/Femology/warden-sdk/pull/8)). `v0.4.0` added the
  Phase 18 "Explain this" schema/validation module covered below
  ([warden-sdk#9](https://github.com/Femology/warden-sdk/pull/9)).
{% endhint %}

## Configure a client

```ts
import { WardenClient } from 'warden-sdk';

const client = new WardenClient({
  contractId: 'CD5QU2E6LOKFAZFESIZSAA4IENH5SZHJVU4Y6532WNZSXPZDYRKEEVUW',
  rpcUrl: 'https://soroban-testnet.stellar.org',
  networkPassphrase: 'Test SDF Network ; September 2015',
  referenceAssetDecimals: 7,
});
```

`referenceAssetDecimals` is `7` because the reference asset for this deployment is
native XLM's Stellar Asset Contract — a config value, not something the SDK looks up
over the network.

## Example 1 — read a policy and velocity window

This is a genuine read against the live contract, run exactly as written, against a
brand-new funded Testnet wallet that has never called `set_policy`:

```ts
const wallet = 'GCINF4I5LDWCW2ZLJKQEBOGFBACQNUETMPN6WYIEDPSFPTOTWLPCCPRC';

const policy = await client.getPolicy(wallet);
console.log(policy);
```

**Actual output:**

```
null
```

`getPolicy` returns `null` (not an error) if the wallet hasn't set one yet — check for
that before assuming a policy exists. (`v0.2.0` got this wrong in practice — see the
hint above.)

## Example 2 — set a policy (a write, signed and submitted)

Mutating calls follow a build → sign → submit pattern: `warden-sdk` never signs
anything itself. This example signs with a plain classic Stellar keypair, the simplest
case (a real app signs via `passkey-kit` or another wallet signer instead — see
[`warden-app`](https://github.com/Femology/warden-app) for that full flow).

Note `PortablePolicyRule.trustedRecipients` in the shape below: it's there for symmetry
with the `Policy` type you read back, but `set_policy` itself has no trusted-recipients
parameter at all (see [Contract reference](contract-reference.md#set_policy)) — the
contract preserves whatever's already stored and manages it only through
`add_trusted_recipient`/`remove_trusted_recipient`. Passing anything here is a no-op.

```ts
import { Keypair, TransactionBuilder } from '@stellar/stellar-sdk';

const NETWORK_PASSPHRASE = 'Test SDF Network ; September 2015';
const keypair = Keypair.fromSecret(process.env.DEPLOYER_SECRET);
const wallet = keypair.publicKey();

function sign(xdr: string): string {
  const tx = TransactionBuilder.fromXDR(xdr, NETWORK_PASSPHRASE);
  tx.sign(keypair);
  return tx.toXDR();
}

const { xdr } = await client.buildSetPolicy(wallet, {
  version: 1,
  maxAmountNoStepUp: '150.00',
  dailyVelocityCap: '500.00',
  hourlyVelocityCap: '200.00',
  newRecipientRequiresStepUp: true,
  trustedRecipients: [],
  trustDecaySeconds: 2_592_000, // 30 days
});

await client.submitSetPolicy(sign(xdr));
console.log('set_policy submitted.');
```

**Actual output:**

```
set_policy submitted.
```

Followed by a real `getPolicy` read right after:

```json
{
  "owner": "GCINF4I5LDWCW2ZLJKQEBOGFBACQNUETMPN6WYIEDPSFPTOTWLPCCPRC",
  "maxNoStepUp": "150",
  "dailyVelocityCap": "500",
  "hourlyVelocityCap": "200",
  "newRecipientRequiresStepUp": true,
  "trustedRecipients": {},
  "trustDecaySeconds": "2592000",
  "updatedAt": "1789205247"
}
```

`trustedRecipients` is empty because this is a brand-new policy — trusting a recipient
is a separate call, next.

## Example 3 — trust a recipient, then evaluate transfers

```ts
const recipient = 'GA7QYNF7SOWQ3GLR2BGMZEHXAVIRZA4KVWLTJJFC7MGXUA74P7UJVSGZ';

const { xdr } = await client.buildAddTrustedRecipient(wallet, recipient);
await client.submitAddTrustedRecipient(sign(xdr));
console.log('add_trusted_recipient submitted.');
```

**Actual output**, followed by a `getPolicy` read:

```
add_trusted_recipient submitted.
```

```json
{
  "owner": "GCINF4I5LDWCW2ZLJKQEBOGFBACQNUETMPN6WYIEDPSFPTOTWLPCCPRC",
  "maxNoStepUp": "150",
  "dailyVelocityCap": "500",
  "hourlyVelocityCap": "200",
  "newRecipientRequiresStepUp": true,
  "trustedRecipients": {
    "GA7QYNF7SOWQ3GLR2BGMZEHXAVIRZA4KVWLTJJFC7MGXUA74P7UJVSGZ": "1789205257"
  },
  "trustDecaySeconds": "2592000",
  "updatedAt": "1789205257"
}
```

The value against the recipient is `last_paid_at` (unix seconds) — the timestamp
[trust decay](protocol-mechanics.md#trust-decay) measures against.

Now evaluate a transfer:

```ts
const { xdr } = await client.buildEvaluate(wallet, recipient, '50.00');
const decision = await client.submitEvaluate(sign(xdr));
console.log(decision);
```

**Actual output:**

```json
{ "type": "Allow" }
```

Run again with an amount over the wallet's 150 no-confirmation limit:

```ts
const { xdr } = await client.buildEvaluate(wallet, recipient, '200.00');
const decision = await client.submitEvaluate(sign(xdr));
console.log(decision);
```

**Actual output:**

```json
{ "type": "RequireStepUp", "reason": "AmountExceeded" }
```

See [Protocol mechanics](protocol-mechanics.md) for the full real sequence, including
`HourlyVelocityExceeded` and `VelocityExceeded` results.

## Handling errors

Every `submit*` method throws a `WardenSdkError`, never a bare string:

```ts
import { WardenSdkError } from 'warden-sdk';

try {
  await client.submitSetPolicy(signedXdr);
} catch (error) {
  if (error instanceof WardenSdkError) {
    console.log(error.code, error.message);
  }
}
```

## Environment variables

If you're wiring Warden into an app rather than a script, these are the values every
piece needs — the exact names used across `warden-app` and `warden-monitor`:

| Variable | Purpose |
|---|---|
| `WARDEN_CONTRACT_ID` / `NEXT_PUBLIC_WARDEN_CONTRACT_ID` | `CD5QU2E6LOKFAZFESIZSAA4IENH5SZHJVU4Y6532WNZSXPZDYRKEEVUW` |
| `WARDEN_RPC_URL` / `NEXT_PUBLIC_WARDEN_RPC_URL` | `https://soroban-testnet.stellar.org` |
| `WARDEN_NETWORK_PASSPHRASE` / `NEXT_PUBLIC_WARDEN_NETWORK_PASSPHRASE` | `Test SDF Network ; September 2015` |
| `WARDEN_REFERENCE_ASSET` / `NEXT_PUBLIC_WARDEN_REFERENCE_ASSET` | `CDLZFC3SYJYDZT7K67VZ75HPJVIEUVNIXF47ZG2FB2RMQQVU2HHGCYSC` |
| `WARDEN_DEPLOY_LEDGER` | `4635844` — an indexer's starting point; wrong or missing, it either rescans from genesis or misses early events |

{% hint style="warning" %}
In any Next.js app (`warden-app`, `warden-monitor`'s dashboard), `NEXT_PUBLIC_*`
  variables are inlined into the JavaScript bundle **at build time**, not read at
  runtime. Changing one and restarting the server does nothing — you need a new build.
  See `warden-monitor`'s
  [`HOSTING.md`](https://github.com/Femology/warden-monitor/blob/main/HOSTING.md) for
  the full explanation of this failure mode.
{% endhint %}

## Money is always a decimal string

Every amount in and out of `warden-sdk` is a string like `"150.00"`, never a JS
`number`. Converting to and from the on-chain `i128` is exact fixed-point string
arithmetic — it never routes through `parseFloat` or `Number`. Don't parse an amount
yourself; pass the string straight through.

## Phase 18 — "Explain this"

`warden-sdk` v0.4.0 added `buildExplainPrompt`, `validateExplanationResponse`, and
`fallbackExplanation` — pure, secret-free helpers for the "Explain this" feature both
`warden-app` and `warden-monitor` build a server route around. This is deliberately
**not** an SDK function that calls a model itself; the actual network call (and the
model API key) belongs entirely in each app's own server-only route, never in this
library, since `warden-sdk` is also imported by client-side code and a key must never
end up reachable there by accident.

```ts
import { buildExplainPrompt, validateExplanationResponse, fallbackExplanation } from 'warden-sdk';
import type { ExplanationInput } from 'warden-sdk';

const input: ExplanationInput = {
  eventType: 'stepup_required',
  reason: 'AmountExceeded',
  amount: '200.00',
};

const { system, user } = buildExplainPrompt(input); // user is JSON.stringify(input), nothing else
// ...call your model provider with `system`/`user`, then:
const explanation = validateExplanationResponse(modelResponseJson, input);
const result = explanation ?? fallbackExplanation(input); // never render a null result
```

`validateExplanationResponse` enforces the fixed schema (`summary`, `factors`,
`next_steps`) exactly, and rejects a response that mentions a `StepUpReason` other than
the one actually in `input` — the model hallucinating a reason code it was never given.
Any deviation returns `null`; always fall back to `fallbackExplanation(input)` rather
than rendering a null result. See `warden-app`'s and `warden-monitor`'s own
`/api/explain` route source for a complete, real implementation calling Evomap
(`evomap-deepseek-v4-flash`), including the fallback-on-any-failure behavior.
