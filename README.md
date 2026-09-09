# hci-codec-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

The Bluetooth HCI packet layer with no radio attached: commands, events
and ACL data to and from bytes (Core Vol 4 Part E), the H4 type-byte
framing that lets the three share one UART (Vol 4 Part A § 2), and the
deframer that stands between "some bytes turned up" and "here is one
packet".

It knows nothing about transports and nothing about what any packet
means.  A host stack drives it over a UART; a controller drives it over
an in-RAM ring; a packet-capture tool drives it over a file.  All three
get the same code, because the package has no idea which one it is.

## Adding it, and checking it

```bash
novo pkg add hci-codec-nv    # into your novo.toml
novo pkg build               # type- and effect-check the package
novo test tests/hci_codec_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion below the first constants fails with `not implemented:
hci_codec.<fn>`.  They turn green one at a time as bodies land.

## The one example that will work

```novo
use hci_codec

// A UART handed us some bytes.  Feed them, take what came out.
fn on_bytes(d: Deframer, chunk: [u8]) -> Deframer
    var cur = d
    var step = hci_codec.feed(cur, chunk)
    var go = true
    while go
        match step.packet
            None    => go = false
            Some(p) =>
                handle(p)
                step = hci_codec.take(step.deframer)
        cur = step.deframer
    cur

fn handle(framed: [u8])
    match hci_codec.decode(framed)
        Err(_)  => nothing()
        Ok(pkt) =>
            match pkt
                Evt(e) => on_event(e)
                Acl(a) => on_acl(a)
                Cmd(_) => nothing()
```

## The layer, and why

`core`.  Everything here is arithmetic over bytes the caller already
holds — nothing is read, nothing is written, and the deframer's state is
a value the caller owns rather than a buffer the package hides.  That is
what lets the same deframer run in a controller's receive interrupt and
in a host tool, which is the reason to split it out of a stack at all.

`tests/embedded_probe.nv` is that claim in a form that either builds or
does not.  **It does not build today**, and the reason is filed rather
than designed around: the `Error` trait is absent from the prelude at
`@tier(embedded)`, so `Result<T, DecodeError>` cannot be spelled for a
device — and neither can `Result<T, Str>`, which the compiler's own hint
recommends.  That is an open toolchain defect
(`result-is-unusable-at-tier-embedded-no-error-trait`), not a property
of this package, and the signatures keep the `Result` rather than
retreating to `?T`.

## The load-bearing interface

Two types, and the fact that they are two.

```novo
pub enum Scan
    NeedMore
    Resync
    Complete(length: Int)

pub enum DecodeError
    TooShort(have: Int, need: Int)
    LengthMismatch(declared: Int, actual: Int)
    UnsupportedPacketType(kind: PacketType)
```

**"Wait for more bytes" is not an error, and this package will not let
you conflate the two.**  A hand-written HCI stack usually has one
integer channel for both — a negative length meaning "need more", a
different negative meaning "bad type byte" — and every caller then has
to remember which sentinel is which.  `Scan` is the answer to "how much
of this buffer is a packet", asked of a partial stream where being
incomplete is the normal case.  `DecodeError` is the answer to "this
packet arrived whole and is wrong", which is a controller fault or a
framing loss and not something to wait out.

The second load-bearing shape is `DeframeStep`, and it is a struct rather than
a tuple on purpose:

```novo
pub struct DeframeStep
    deframer: Deframer
    packet: ?[u8]
```

A tuple's components come out only through `match`, which reads badly
for a value whose two halves are used in different places — the caller
reassigns one and dispatches on the other.  The reference stack reached
the same conclusion independently and for the same reason.

## Taking a stream from the host

`drain` is the one function with an effect row, and the row is a
parameter rather than an effect:

```novo
pub trait ByteSource[e]
    fn read_byte(self) -> ?Int [e]

pub fn drain<S: ByteSource[e]>(d: Deframer, src: S) -> Drained [e]
```

It costs whatever the source costs — `[io]` against a host's UART,
nothing against a buffer of captured bytes — which is what lets a `core`
package offer the read-until-empty loop instead of making every caller
write it (SPEC § 5.6, and `docs/publishing.md` calls this the direct
shape for a core package taking a stream).

**It has no test in this release**, and its absence is the second filed
defect: calling an effect-polymorphic function from another module
specialises it, and the specialised copy keeps `[e]` while losing the
bound that bound it
(`effect-polymorphic-fn-loses-its-bound-when-monomorphised-across-modules`).
Since a package's `pub fn` is by construction called from another
module, the shape does not work anywhere it was meant to be used.  The
signature stays as it is; a package that dropped it to make its own test
file compile would be reporting a defect as a design.

## The reference implementation

`orbit/ble`'s own HCI layer, spread across five modules that this
package makes one: `common/hci_frame.nv` (the headers and the typed
messages), `common/hci_commands.nv` (opcodes), `host/hci_codec.nv`
(command builders and event accessors), `host/hci_h4.nv` (framing and
the length rule) and `host/hci_deframer.nv` (feed and take).  The
Bluetooth Core Specification Vol 4 Parts A and E are the source of the
test vectors.

Three things change in the port, and each is a thing the reference could
not have and this package can.  The `-1` / `-2` sentinels from
`h4_packet_length` become `Scan`.  The `PKT_COMMAND` … `PKT_ISO`
integers become `PacketType`, so a type byte cannot be compared against
an event code by accident.  And the twenty-odd `opcode_le_*()` accessor
functions become `opcode(ogf, ocf)` plus the two group constants: the
reference wrote a function per opcode because a cross-module `const` was
not reachable at the time, and that constraint is gone.

## Status

| item | implemented |
| --- | --- |
| the `OGF_*` and `EVT_*` constants | yes — they are constants |
| `hci_codec.h4_type_byte`, `.h4_packet_type`, `.h4_frame`, `.h4_scan` | no |
| `hci_codec.opcode`, `.opcode_ogf`, `.opcode_ocf` | no |
| `hci_codec.encode_command`, `.decode_command` | no |
| `hci_codec.encode_event`, `.decode_event` | no |
| `hci_codec.encode_acl`, `.decode_acl` | no |
| `hci_codec.decode` | no |
| `hci_codec.command_complete_opcode`, `.command_complete_status` | no |
| `hci_codec.deframer`, `.feed`, `.feed_byte`, `.take`, `.pending_len` | no |
| `hci_codec.drain` | no |
| `hci_codec.DecodeError.message` | no |
