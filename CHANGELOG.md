# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.1.0 — 2026-10-10

The **implementation**. Every body the 0.0.4 interface published is
written, `novo test` is green over two suites, and the lines under
`src/` are all covered by them, with no `cov: skip` marker anywhere.

- **The whole package is embedded-clean, with nothing carved out.**
  A device build stamps `@tier(embedded)` on every function of every
  module it compiles, reachable or not, so a `Result` and a
  payload-carrying variant are built inside a `system{}` block — a heap
  cell the tier admits only there — and a buffer is copied byte by
  byte, because the tier has no slice.  Each block delivers with an
  explicit `return`, because a tail expression in one is discarded by
  novo 0.20.1 — filed as
  `a-system-block-in-value-position-discards-its-value`, fixed in 0.20.2
  for a function's own tail, so the `return`s stay only until
  `a-system-block-whose-value-is-a-result-call-is-refused-at-a-device-boundary`
  closes.
- **`drain` streams, and streaming is what kept it on that tier.** It
  pulls until one packet completes and answers the `DeframeStep` the
  byte-at-a-time entry points answer, so nothing accumulates and the
  declaration carries no tier exception. The 0.0.4 entry has the shape
  it replaced, and why that shape could not be built.
- **`drain` binds its source to a `var` name before reading it.** A
  source that consumes a buffer has to advance its own cursor, which
  the trait admits as a `var self` receiver, and a `var self` method
  cannot be called through a plain parameter.
- **A scan answers `Complete` only when every byte the length counts
  has arrived.** `h4_scan` is the length rule, so a caller that slices
  by its answer never runs off the end, and a prefix stays `NeedMore`
  however often it is scanned.
- **SCO, ISO and a byte that names no layer all answer `Resync`**, and
  `take` drops exactly one byte on one: the deframer never hands
  `decode` a layer that function would refuse, and a caller that keeps
  calling resynchronises without a rule of its own.
- **`decode` answers `TooShort(have: 0, need: 1)` for an empty buffer
  and for a byte that names no layer.** No `PacketType` names an
  unknown byte, and a `DecodeError` describes a packet that arrived
  whole and was wrong, so neither fault can be an error of that kind.
- **The ACL boundary bits 0b11 read as `FirstNonFlushable`.** They are
  BR/EDR's, which has no LE meaning; a fragment that arrived whole is
  still a fragment, and refusing it would throw away the payload around
  it.

The new tests live in `tests/hci_codec_edge_tests.nv` — every layer's
type byte, the guards that answer `None`, the text of a refusal, the
boundary bits no round trip reaches, and `drain` over a source of its
own — because the API suite is the published oracle and is not edited.

The two toolchain defects the 0.0.1 entry recorded are gone.
`tests/embedded_probe.nv` builds for the nRF52, because a device build
prepends `error.nv` and so `Result<T, DecodeError>` is spellable there.
And `drain` is called from another module by the new suite, because the
specialised copy now keeps the bound that binds its `[e]`: so `drain`
has tests after all.

## 0.0.4 — 2026-10-10

The **interface**, corrected. `drain` answers a `DeframeStep` rather
than a `Drained`, and `Drained` is gone. This is the release the 0.1.0
implements; every other declaration is as 0.0.3 published it.

The shape it replaces could not be built. `drain` answered
`Drained { deframer, packets: [[u8]] }` — every packet a source
yielded, collected — and collecting an unbounded number of
variable-length frames is not something a device can do. A device
build refused it five ways at once: a list literal, an unbounded
`while`, `list.push` and `list.clone`, which are not on the embedded
surface, and the boxed struct literal. A `system{}` block discharges
the first and the last, and `while true` is one of the three loop
shapes that tier admits — but `list.push` is the growing half of
`list`, which the tier refuses outright, and the bounded alternative,
`with_capacity` plus `try_push`, wants a capacity and `@value`
elements, and a queue of frames has neither.

So the claim and the signature contradicted each other: the package
promised a device tier, and `drain` could not be written for one. The
fix is the shape the package already had one function away —
`feed_byte` answers one packet at a time through `DeframeStep`, so
`drain` became the loop around it, and a caller that wants every packet
loops until a call answers none.

## 0.0.3 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

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

### Design notes

Moved here from the README, which now states only what a user needs.

- `DeframeStep` is a struct rather than a tuple because a tuple's
  components come out only through `match`, which reads badly for a
  value whose two halves are used at different places: the caller
  reassigns the deframer and dispatches on the packet. The reference
  stack reached the same conclusion independently.
- The port collapses five modules of `orbit/ble` into one:
  `common/hci_frame.nv` (the headers and the typed messages),
  `common/hci_commands.nv` (opcodes), `host/hci_codec.nv` (command
  builders and event accessors), `host/hci_h4.nv` (framing and the
  length rule) and `host/hci_deframer.nv` (feed and take).
- Three things change in the port. The `-1` and `-2` sentinels of
  `h4_packet_length` become `Scan`. The `PKT_COMMAND` … `PKT_ISO`
  integers become `PacketType`, so a type byte cannot be compared
  against an event code by accident. The twenty-odd `opcode_le_*()`
  accessor functions become `opcode(ogf, ocf)` plus the group
  constants, because a cross-module `const` was not reachable when the
  reference was written and now is.
