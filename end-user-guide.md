# Using Warden

This page is for you if you're using a wallet built on Warden, not building one. No
code, no jargon.

## Setting your limits

Open the policy screen in your wallet app. You'll set three things:

- **An amount below which nothing extra is asked of you.** Send under this amount, and
  your usual confirmation (your passkey, your face, your fingerprint) is all that's
  needed. There's no second check.
- **A daily limit.** The total you can send across all transfers in a rolling 24-hour
  window before you're asked to confirm again, even if each individual transfer is
  small.
- **Whether new recipients need extra confirmation.** Turn this on, and the first time
  you send to an address you haven't paid before, you'll be asked to confirm. After
  that, if you mark them as trusted, future transfers to them skip this check.

There's no single "right" setting — it depends on how you use your wallet. Someone
making frequent small payments to the same few people might set a low per-transfer
amount but trust everyone they pay regularly. Someone who rarely sends money might
leave the new-recipient check on permanently and set a low daily limit.

## Trusted recipients

Once you've sent to someone and want future transfers to them to skip the
new-recipient check, add them as trusted. It's the same idea as saving a contact.

If you no longer want to trust someone — you don't send to them anymore, or you're not
sure the address is right — remove them. Nothing about your policy's amount or daily
limit changes; only that one recipient's status.

## What a step-up prompt means

When a transfer needs one more confirmation, you'll see a clear reason, not a generic
warning:

- **"This amount is above your no-confirmation limit."** The transfer itself is larger
  than what you set to go through automatically.
- **"You haven't sent to this recipient before."** This address isn't on your trusted
  list yet.
- **"This would put you over your daily limit."** You're close to (or over) your
  24-hour spending cap — even if this specific transfer is small on its own.

{% hint style="info" %}
A step-up isn't an error, and it isn't the app malfunctioning. It's your own rule
  working exactly as you set it. If you didn't want that check, change your policy —
  don't fight it every time it appears.
{% endhint %}

Confirming a step-up completes the transfer. Cancelling it stops the transfer entirely
— nothing partial happens, and it doesn't count against you differently than if you'd
never started it.

## A worked example

Say you set your no-confirmation amount to $150 and your daily limit to $500, with new
recipients requiring confirmation. Here's what actually happens across a day of real
transfers to a friend you've already trusted:

- **$50** → goes straight through. Under your limit, recipient is trusted.
- **$200** → asks you to confirm. It's over your $150 no-confirmation amount — this is
  checked before your daily limit, so it doesn't matter how much room you have left in
  your $500 cap; a single transfer over $150 always asks.
- **$260** → asks you to confirm, same reason: it's over $150 on its own.
- **$10**, later that day, after the above → asks you to confirm. This $10 is well
  under your $150 limit, but your running total for the day is already past $500, so
  even a small transfer now needs confirmation.

Every one of those transfers — including the three that asked for confirmation — counts
toward your daily total once you complete it. That's deliberate: a transfer that
happened still happened, whether or not it needed an extra tap.

## If you're integrating Warden into a fintech app

See the [Developer guide](developer-guide.md).
