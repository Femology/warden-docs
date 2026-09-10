# Developer guide

{% hint style="warning" %}
Every code sample on this page was run against the real deployed contract
  (`CBFQ752LFNC57U4KWDAEKNU43PLBWJ7M2B4ZRYUMCWL62JHJNUYJVMB5` on Testnet) while writing
  this guide. The output shown is real, not invented.
{% endhint %}

## Install

```bash
npm install warden-sdk@github:Femology/warden-sdk#v0.1.3
```

`warden-sdk` isn't published to npm yet (tracked in
[warden-sdk#2](https://github.com/Femology/warden-sdk/issues/2)) — install it as a git
dependency pinned to a tag, as shown above. ESM only, Node ≥18.

{% hint style="info" %}
Pin `v0.1.3` specifically, not an earlier tag. `v0.1.2` has a real bug where every
  `submit*` call throws `"The transaction has not yet been signed"` against a live
  network, regardless of whether you signed correctly — found and fixed while writing
  this guide. See [warden-sdk#5](https://github.com/Femology/warden-sdk/pull/5).
{% endhint %}

## Configure a client

```ts
import { WardenClient } from 'warden-sdk';

const client = new WardenClient({
  contractId: 'CBFQ752LFNC57U4KWDAEKNU43PLBWJ7M2B4ZRYUMCWL62JHJNUYJVMB5',
  rpcUrl: 'https://soroban-testnet.stellar.org',
  networkPassphrase: 'Test SDF Network ; September 2015',
  referenceAssetDecimals: 7,
});
```

`referenceAssetDecimals` is `7` because the reference asset for this deployment is
native XLM's Stellar Asset Contract — a config value, not something the SDK looks up
over the network.

## Example 1 — read a policy and velocity window

This is a genuine read against the live contract, run exactly as written:

```ts
const wallet = 'GCZLMMKEOPOG5OB5QRLGH5ZKG7ACQNKX7KTT6UTXPFHUPS7FFSFFU5YM';

const policy = await client.getPolicy(wallet);
console.log(policy);

const velocity = await client.getVelocity(wallet);
console.log(velocity);
```

**Actual output:**

```json
{
  "owner": "GCZLMMKEOPOG5OB5QRLGH5ZKG7ACQNKX7KTT6UTXPFHUPS7FFSFFU5YM",
  "maxNoStepUp": "150",
  "dailyVelocityCap": "500",
  "newRecipientRequiresStepUp": true,
  "trustedRecipients": [
    "GA7QYNF7SOWQ3GLR2BGMZEHXAVIRZA4KVWLTJJFC7MGXUA74P7UJVSGZ"
  ],
  "updatedAt": "1789057652"
}
{
  "windowStart": "1789052887",
  "cumulativeAmount": "520.00001",
  "txCount": 5
}
```

`getPolicy` returns `null` (not an error) if the wallet hasn't set one — check for that
before assuming a policy exists.

## Example 2 — set a policy (a write, signed and submitted)

Mutating calls follow a build → sign → submit pattern: `warden-sdk` never signs
anything itself. This example signs with a plain classic Stellar keypair, the simplest
case (a real app signs via `passkey-kit` or another wallet signer instead — see
[`warden-app`](https://github.com/Femology/warden-app) for that full flow).

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
  newRecipientRequiresStepUp: true,
  trustedRecipients: [],
});

await client.submitSetPolicy(sign(xdr));
console.log('set_policy submitted.');
```

**Actual output:**

```
set_policy submitted.
```

Followed by a `getPolicy` call showing the values took effect — exactly the JSON shown
in Example 1 above, since that read was captured right after this write.

## Example 3 — evaluate a transfer and read the decision

```ts
const recipient = 'GA7QYNF7SOWQ3GLR2BGMZEHXAVIRZA4KVWLTJJFC7MGXUA74P7UJVSGZ';

const { xdr } = await client.buildEvaluate(wallet, recipient, '50.00');
const decision = await client.submitEvaluate(sign(xdr));
console.log(decision);
```

**Actual output**, run against a wallet that had already trusted this recipient:

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

See [Protocol mechanics](protocol-mechanics.md) for the full real sequence, including a
`VelocityExceeded` result.

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
| `WARDEN_CONTRACT_ID` / `NEXT_PUBLIC_WARDEN_CONTRACT_ID` | `CBFQ752LFNC57U4KWDAEKNU43PLBWJ7M2B4ZRYUMCWL62JHJNUYJVMB5` |
| `WARDEN_RPC_URL` / `NEXT_PUBLIC_WARDEN_RPC_URL` | `https://soroban-testnet.stellar.org` |
| `WARDEN_NETWORK_PASSPHRASE` / `NEXT_PUBLIC_WARDEN_NETWORK_PASSPHRASE` | `Test SDF Network ; September 2015` |
| `WARDEN_REFERENCE_ASSET` / `NEXT_PUBLIC_WARDEN_REFERENCE_ASSET` | `CDLZFC3SYJYDZT7K67VZ75HPJVIEUVNIXF47ZG2FB2RMQQVU2HHGCYSC` |
| `WARDEN_DEPLOY_LEDGER` | `4598184` — an indexer's starting point; wrong or missing, it either rescans from genesis or misses early events |

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
