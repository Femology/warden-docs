# Using Warden

This page is for you if you're using a wallet built on Warden, not building one. No
code, no jargon.

## Setting your limits

Open the policy screen in your wallet app. You'll set:

- **An amount below which nothing extra is asked of you.** Send under this amount, and
  your usual confirmation (your passkey, your face, your fingerprint) is all that's
  needed. There's no second check.
- **A daily limit.** The total you can send across all transfers in a rolling 24-hour
  window before you're asked to confirm again, even if each individual transfer is
  small.
- **An hourly limit.** The same idea, but tighter and over a shorter window — it catches
  a burst of transfers that's fast enough to look suspicious even while you're nowhere
  near your daily total. It can't be set looser than your daily limit (that would make
  it pointless), but it can be — and usually should be — noticeably tighter.
- **Whether new recipients need extra confirmation.** Turn this on, and the first time
  you send to an address you haven't paid before, you'll be asked to confirm. After
  that, if you mark them as trusted, future transfers to them skip this check.
- **How long trust lasts without a payment.** A trusted recipient you haven't paid in
  this long goes back to being treated as new — see [Trusted recipients](#trusted-recipients)
  below.

There's no single "right" setting — it depends on how you use your wallet. Someone
making frequent small payments to the same few people might set a low per-transfer
amount but trust everyone they pay regularly, with a generous hourly limit to match how
they actually send. Someone who rarely sends money might leave the new-recipient check
on permanently and set tight hourly and daily limits.

## Trusted recipients

Once you've sent to someone and want future transfers to them to skip the
new-recipient check, add them as trusted. It's the same idea as saving a contact.

Trust isn't permanent, though. If you haven't paid a trusted recipient in longer than
the period you set (30 days is a common choice), they quietly go back to needing
confirmation next time — the same as if you'd never trusted them. One more completed
transfer to them re-establishes trust from that point forward. This isn't a bug or a
glitch: it's there so "trusted" keeps meaning "someone you deal with," not "someone you
paid once, years ago, and forgot about."

If you no longer want to trust someone — you don't send to them anymore, or you're not
sure the address is right — remove them outright. Nothing about your policy's amounts
or limits changes; only that one recipient's status.

## What a step-up prompt means

When a transfer needs one more confirmation, you'll see a clear reason, not a generic
warning:

- **"This amount is above your no-confirmation limit."** The transfer itself is larger
  than what you set to go through automatically.
- **"You haven't sent to this recipient before."** Either this address isn't on your
  trusted list yet, or it used to be and your trust in it has expired from disuse.
- **"This would put you over your hourly limit."** You're sending fast enough right now
  to clear your shorter-window cap — even if you're nowhere near your daily one.
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

Say you set your no-confirmation amount to $150, your hourly limit to $200, your daily
limit to $500, with new recipients requiring confirmation. This is a real sequence, not
a made-up one — here's what actually happens across an hour of transfers to a friend
you've already trusted:

- **$50** → goes straight through. Under your limit, recipient is trusted, plenty of
  room in both your hourly and daily caps.
- **$200** → asks you to confirm. It's over your $150 no-confirmation amount — this is
  checked before either velocity limit, so it doesn't matter how much room you have
  left in your caps; a single transfer over $150 always asks.
- **$100**, right after → asks you to confirm again, but for a different reason this
  time: on its own, $100 is well under your $150 limit, but your running total for the
  last hour is already $250 (the $50 plus the $200, since a transfer that needed
  confirmation still counts once you complete it) — add $100 and you're at $350,
  over your $200 hourly cap.
- **$100**, again → same reason. Your hourly total keeps climbing.

Notice your $500 daily cap never actually came into play here — you're nowhere near it.
It was your tighter *hourly* limit that caught this burst of transfers, which is
exactly what it's there for: your daily limit protects your whole day, your hourly
limit protects any given hour from moving faster than you're comfortable with.

Every one of those transfers — including the three that asked for confirmation — counts
toward both your hourly and daily totals once you complete it. That's deliberate: a
transfer that happened still happened, whether or not it needed an extra tap.

## If you're integrating Warden into a fintech app

See the [Developer guide](developer-guide.md).
