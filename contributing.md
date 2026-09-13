# Contributing

Warden is four coordinated repos, each with its own `CONTRIBUTING.md` covering local
setup, commit conventions, and code style specific to that repo's stack:

- **[warden-contract](https://github.com/Femology/warden-contract/blob/main/CONTRIBUTING.md)** -- Rust / Soroban. The policy engine itself.
- **[warden-sdk](https://github.com/Femology/warden-sdk/blob/main/CONTRIBUTING.md)** -- TypeScript. The client library, never signs anything.
- **[warden-app](https://github.com/Femology/warden-app/blob/main/CONTRIBUTING.md)** -- Next.js. The reference wallet, passkey-signed.
- **[warden-monitor](https://github.com/Femology/warden-monitor/blob/main/CONTRIBUTING.md)** -- Node + Next.js. Read-only observability.

## The rule that spans every repo

**The allow/step-up decision exists in exactly one place: `warden-contract`.** A change
to `warden-sdk`, `warden-app`, or `warden-monitor` that would let any of them make or
override that decision is out of scope, no matter how convenient it seems. This is the
one rule none of the four repos' own `CONTRIBUTING.md` files compromise on.

## Where to start

Each repo's issue tracker has a set of labeled, scoped next-step issues; search for
`complexity: small` if you want something you can finish in an afternoon. A few
examples of what's currently open:

- [warden-contract#3](https://github.com/Femology/warden-contract/issues/3): the
  velocity window's fixed-reset limitation, described on
  [Protocol mechanics](protocol-mechanics.md).
- [warden-app#2](https://github.com/Femology/warden-app/issues/2): deploying
  `warden-app` to a real public URL.
- [warden-monitor#2](https://github.com/Femology/warden-monitor/issues/2) and
  [#3](https://github.com/Femology/warden-monitor/issues/3): deploying the indexer and
  dashboard.

## Reporting a security issue

**Don't open a public issue for a security vulnerability.** Each repo has its own
`SECURITY.md` with a private contact and scope. Start there:
[warden-contract](https://github.com/Femology/warden-contract/blob/main/SECURITY.md) ·
[warden-sdk](https://github.com/Femology/warden-sdk/blob/main/SECURITY.md) ·
[warden-app](https://github.com/Femology/warden-app/blob/main/SECURITY.md) ·
[warden-monitor](https://github.com/Femology/warden-monitor/blob/main/SECURITY.md).

Every one of these states the same thing plainly: **Warden is unaudited.** Don't build
a production integration handling real funds on any of these repos without an
independent security review first.
