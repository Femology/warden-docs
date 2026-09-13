# Introduction

## The problem

Most wallet apps treat every payment the same. Send $2 to your sister and you get the
same identity check as sending $2,000 to a stranger. That's not caution, it's just
undifferentiated friction, and users notice.

Here's a real complaint, verbatim, from a live Nigerian fintech's app-store reviews:

> "why do I have to scan my face for every single action on the app… there is no
> setting where you can change it. I had to move all my funds out immediately."

That's not a hypothetical UX nitpick. That's a user closing their account over
authentication friction that didn't distinguish a routine transfer from a risky one.
Meanwhile, the opposite failure (*no* friction on a large transfer to a brand-new
address) is how account-takeover losses actually happen. Warden is built for the
space between those two failures: the missing middle, where friction shows up because
the risk is real, not because the app can't tell the difference.

## What Warden is

Warden is an open-source risk-policy engine for Stellar smart wallets. For every
transfer, it decides whether the wallet's ordinary signature is enough, or whether the
transfer needs one more explicit confirmation, a **step-up**. That decision is based
on four things, checked in this order:

1. **Is the recipient flagged?** A small, admin-managed registry of known-bad
   addresses. If the recipient is on it, step-up is required outright, regardless of
   everything else below.
2. **The amount.** Is it under the threshold you've set for "just let it through"?
3. **The recipient.** Have you sent to them recently, or trusted them explicitly?
   Trust itself fades if you haven't paid someone in a while.
4. **Velocity.** How much have you already sent in the last hour, and the last 24
   hours? Two independent windows, so a fast burst is caught even when the day's total
   is nowhere near its cap.

{% hint style="info" %}
Step-up is not an error. It's not a rejection, and it's not the app being broken.
  It's the system working correctly: friction arriving exactly where it should.
{% endhint %}

Two more things exist alongside the decision itself: a **guardian recovery**
subsystem (a wallet owner names guardians who can, together and only after a
timelock, restore a restricted account without the owner's own signature; see
[`WARDEN-PROTOCOL.md`](https://github.com/Femology/warden-contract/blob/main/WARDEN-PROTOCOL.md)
for the full state machine), and an **"Explain this"** feature that turns any
step-up's reason code into a plain-language explanation, grounded strictly in that
event's own on-chain facts and validated against a fixed schema before it's ever shown.
See the [Developer guide](developer-guide.md#explain-this).

## Why this lives on Stellar, specifically

Soroban smart wallets support multiple signers with distinct roles, evaluated inside
the wallet's own authorization check. That means the allow/step-up decision can live
**on-chain, at the same trust boundary as the wallet itself**, instead of in a backend
service that a wallet provider has to separately build, operate, and trust. That's an
architectural reason to build this on Stellar, not a "blockchain is fast and cheap"
claim.

## How it works, step by step

1. **A wallet owner sets a policy.** The amount below which no extra confirmation is
   needed, an hourly spending cap, a daily spending cap, whether new recipients require
   a step-up the first time, and how long trust lasts without a payment.
2. **The owner can trust specific recipients.** Once trusted, transfers to that address
   skip the new-recipient check, the same idea as saving a payee. That trust fades on
   its own if the recipient goes unpaid past the configured decay period.
3. **A transfer is attempted.** Before it goes through, the wallet calls Warden's
   `evaluate()` function with the recipient and amount.
4. **Warden decides**, checking in order: is this an untrusted (or trust-decayed) new
   recipient? Is the amount over the no-confirmation threshold? Would this push the
   last hour's spend over its cap? Would it push the day's total spend over its cap?
   The first match wins.
5. **If nothing matches, the transfer proceeds** on the wallet's normal signature
   alone. **If something matches**, the wallet's own UI asks for one more explicit
   confirmation before sending, with a plain-language reason, not a generic "are you
   sure?"

See [Protocol mechanics](protocol-mechanics.md) for the exact decision lifecycle with
real worked numbers, or jump straight to [Using Warden](end-user-guide.md) if you're a
wallet owner, not a developer.

## The four repos

Warden is four coordinated repos, each with one job:

| Repo | Job |
|---|---|
| [`warden-contract`](https://github.com/Femology/warden-contract) | The policy engine. Decides. Nothing else is allowed to make or override this decision. |
| [`warden-sdk`](https://github.com/Femology/warden-sdk) | The TypeScript client library. Builds and submits calls. Never signs anything. |
| [`warden-app`](https://github.com/Femology/warden-app) | The reference wallet. Signs via a WebAuthn passkey and shows the three scenarios end to end. |
| [`warden-monitor`](https://github.com/Femology/warden-monitor) | Read-only telemetry. Watches what happened. Never decides. |

## Honest limitations

Stated here plainly, not buried in a footnote. See each repo's own README and
`SECURITY.md` for the full detail:

- **The step-up gate is app-enforced, not yet cryptographically enforced.** `warden-app`
  asks for a decision and refuses to proceed without confirmation in its own UI, but
  nothing on-chain yet stops a modified client from ignoring that answer.
- **Both velocity windows reset on expiry, not a continuous slide.** A wallet could in
  principle spend up to a cap right before a reset and again right after.
- **This project is unaudited.** Every repo says so directly. Don't put real funds at
  risk on this without an independent security review first.

---

*This page is the first page of the published documentation (GitBook Git Sync uses
README.md as the book's landing page) and this repo's own README at the same time. See
[SUMMARY.md](SUMMARY.md) for the full table of contents, or connect this repo at
[app.gitbook.com](https://app.gitbook.com) to publish it live -- no local build step
needed, Git Sync handles it on every push to main.*
