# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.0.2 — 2026-09-10

- **Toolchain floor is 0.8.9**: the bodies and signatures use what 0.8.9 added (`todo()`, a bound effect parameter, the four layers), and the manifest says so instead of letting an older toolchain fail on an undefined function.  No signature changed.

## 0.0.1 — 2026-09-09

The **interface**, before anyone implements it. Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`. Adding this
package works and calling it panics.

- `PacketType`, `HciCommand`, `HciEvent`, `AclData`, `Boundary` and `Packet` —
  the three HCI layers as types rather than as offsets into a byte list.
- `Scan` for "how much of this buffer is a packet" and `DecodeError` for
  "this packet is wrong". They are two types on purpose: waiting for
  more bytes is not an error, and a codec that conflates the two makes
  every caller remember which sentinel meant which.
- `Deframer`, `feed`, `feed_byte`, `take` and `pending_len` — the state
  between a byte stream and a packet, as a value the caller owns.
- `drain`, the one function with an effect row, which binds
  `ByteSource`'s effect parameter and so costs whatever the caller's
  source costs and nothing of its own.

Two toolchain defects travel with the release rather than being designed
around. `tests/embedded_probe.nv` does not build, because the `Error`
trait is absent from the prelude at `@tier(embedded)` and so
`Result<T, DecodeError>` cannot be spelled for a device
(`result-is-unusable-at-tier-embedded-no-error-trait`). And `drain` has
no test, because calling an effect-polymorphic function from another
module specialises it and the specialised copy keeps `[e]` while losing
the bound that bound it
(`effect-polymorphic-fn-loses-its-bound-when-monomorphised-across-modules`).
Both signatures stay as they are.
