# hci-codec-nv

The Host Controller Interface (HCI) is the boundary between the two
halves of a Bluetooth system: the **host**, which is software on an
application processor, and the **controller**, which is the radio and
its firmware. It is specified in the [Bluetooth Core Specification](https://www.bluetooth.com/specifications/specs/core-specification/),
Volume 4. This package turns HCI packets into bytes and bytes into HCI
packets, with no radio and no transport underneath. One package on the
registry is built on it:
[l2cap-nv](https://novo-lang.org/packages/l2cap-nv), which reads and
writes channels inside the ACL data packets this one carries. Below it
is [ble-link-codec-nv](https://novo-lang.org/packages/ble-link-codec-nv),
the link-layer packets a radio puts on the air.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What HCI is

A host asks its controller to do things by sending **commands**, and the
controller answers with **events**. Data for a connection travels in a
third kind of packet, **ACL data**, named after the Asynchronous
Connection-Less logical transport. Core Vol 4 Part E defines all three.
Two further kinds exist, synchronous data for voice and isochronous data
for LE Audio, and this package codes neither.

The three kinds share one wire. **H4** is the transport that lets them:
it puts one byte in front of every packet saying which kind follows.
Core Vol 4 Part A section 2 defines it. The type byte is not part of the
packet, and each kind puts its length field at a different offset and
width.

| Packet | H4 type byte | Header after the type byte | Where the length is |
| --- | --- | --- | --- |
| Command | 0x01 | Two bytes of opcode, one byte of parameter length | One byte at offset 3 |
| ACL data | 0x02 | Two bytes of handle and flags, two of payload length | Two bytes at offset 3 |
| Synchronous data | 0x03 | Not coded by this package | — |
| Event | 0x04 | One byte of event code, one byte of parameter length | One byte at offset 2 |
| Isochronous data | 0x05 | Not coded by this package | — |

Every multi-byte field is little-endian, which is the Bluetooth
convention throughout.

A command is named by a 16-bit **opcode**. The opcode is two fields: a
6-bit **opcode group field** (OGF) that says which family the command
belongs to, and a 10-bit **opcode command field** (OCF) that picks the
command within it. The packing is `ogf << 10 | ocf`, defined in
section 5.4.1. Six groups have constants here.

| OGF | Constant | Family |
| --- | --- | --- |
| 0x01 | `OGF_LINK_CONTROL` | Link control |
| 0x03 | `OGF_CONTROLLER_BASEBAND` | Controller and baseband |
| 0x04 | `OGF_INFORMATIONAL_PARAMETERS` | Informational parameters |
| 0x05 | `OGF_STATUS_PARAMETERS` | Status parameters |
| 0x08 | `OGF_LE_CONTROLLER` | Every Low Energy command |
| 0x3F | `OGF_VENDOR_SPECIFIC` | Vendor commands |

An event is named by a one-byte code, defined in section 7.7. Seven
codes have constants here. Every LE event arrives inside the LE Meta
Event, with a subevent code as the first parameter byte.

| Code | Constant | Event |
| --- | --- | --- |
| 0x05 | `EVT_DISCONNECTION_COMPLETE` | A connection ended |
| 0x08 | `EVT_ENCRYPTION_CHANGE` | Encryption was turned on or off |
| 0x0E | `EVT_COMMAND_COMPLETE` | A command finished, with its return parameters |
| 0x0F | `EVT_COMMAND_STATUS` | A command was accepted and its result comes later |
| 0x10 | `EVT_HARDWARE_ERROR` | The controller reported a fault |
| 0x13 | `EVT_NUMBER_OF_COMPLETED_PACKETS` | The controller's flow-control credit |
| 0x3E | `EVT_LE_META_EVENT` | An LE event, with its subevent code first |

A byte stream does not arrive in packets. A UART read returns whatever
was in the buffer, which may be half a packet or two and a half. The
**deframer** is the state between "some bytes turned up" and "here is
one packet". It is a value the caller holds one of per link, so this
package holds nothing of its own.

This package performs no input or output. It reads no device, writes to
no address and consults no clock. Every function is arithmetic over
bytes the caller already holds, so the same code runs in a host stack on
a laptop, in a controller's receive interrupt, and in a capture tool
reading a file.

## Install

```
novo pkg add hci-codec-nv
```

## Example

```novo
use hci_codec

fn main() [io]
    // HCI_Reset: opcode group 0x03, command 0x003, no parameters.
    let reset = HciCommand { opcode: hci_codec.opcode(0x03, 0x003), params: [] }

    // The command's wire bytes, with the H4 type byte 0x01 in front.
    let framed = hci_codec.h4_frame(HciCommand, hci_codec.encode_command(reset))

    // Ask how much of the buffer is one whole packet, without copying.
    match hci_codec.h4_scan(framed)
        NeedMore    => println("the packet has not all arrived")
        Resync      => println("the leading byte names no packet type")
        Complete(n) => println("${n} bytes are one packet")

    // The bytes a transport handed over. The deframer keeps a prefix
    // and hands back one whole packet at a time.
    let step = hci_codec.feed(hci_codec.deframer(), framed)
    match step.packet
        None => println("waiting for more bytes")
        Some(p) =>
            // Read the packet and dispatch on the layer it belongs to.
            match hci_codec.decode(p)
                Ok(Cmd(c)) => println("command ${c.opcode}")
                Ok(Evt(e)) => println("event ${e.code}")
                Ok(Acl(a)) => println("acl on handle ${a.handle}")
                Err(e)     => println("the packet is wrong: ${e.message()}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a `not implemented: hci_codec.<fn>`
panic. The tests are the specification the implementation will have to
satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `hci_codec` | The whole surface. The H4 type bytes and the length rule, opcode packing and the group constants, the event-code constants, commands, events and ACL data as types, encoding and decoding for each, and a deframer that turns a byte stream into packets. |

## How to choose an entry point

There are three ways to get a packet out of incoming bytes, and they
differ in how much of the buffering you do yourself.

**`h4_scan` reads only the length rule.** It answers how many bytes at
the front of the buffer are one packet, and it copies nothing. Use it
when you hold a receive buffer and slice packets out of it yourself.

**The deframer buffers for you.** `feed` takes whatever the transport
handed over and returns the deframer plus one packet if the bytes
completed one. `take` gets a packet that is already buffered without
adding bytes. Use the pair in a host stack, where reads have no relation
to packet boundaries. Keep one deframer per link.

**`feed_byte` takes one byte.** It is the shape an interrupt handler
has, with a byte in a register and nothing to allocate to pass it.

For turning bytes into a typed packet there are two levels. `decode`
takes one whole H4 packet, type byte and all, and answers a `Packet`
naming its layer. `decode_command`, `decode_event` and `decode_acl` take
one layer's bytes without a type byte, for a caller that already knows
which layer it has.

## The rules a user needs

1. **"Wait for more bytes" is not an error.** `h4_scan` answers
   `NeedMore` for a prefix, and the caller keeps the bytes. A
   `DecodeError` means a packet arrived complete and was wrong, and the
   caller drops it and logs a fault. The two answers are opposite
   instructions, so they are separate types.
2. **`Resync` is a caller's loop, not this package's.** A leading byte
   that names no packet type gives `Scan.Resync`. How far to drop is a
   policy that depends on the link, so this package drops nothing and
   scans again where it is told to. `take` drops exactly one byte, so a
   caller that keeps calling it resynchronises without a rule of its own.
3. **The length a scan reports includes the H4 type byte.** A reset
   command is four bytes: the type byte, two opcode bytes and the
   parameter-length byte (Core Vol 4 Part A section 2).
4. **`encode_command`, `encode_event` and `encode_acl` produce a packet
   without a type byte.** `h4_frame` puts the byte in front. `decode`
   expects the byte to be there and the three layer decoders expect it
   not to be.
5. **The parameter-length byte is not a field you set.** `HciCommand`
   and `HciEvent` carry their parameters, and the encoder writes the
   length from them. A struct holding both could disagree with itself.
6. **`feed` drains one packet per call.** A single UART read can
   complete two. Call `take` in a loop until it returns no packet, and
   read `pending_len` to see what is still held.
7. **A truncated packet is held forever.** This package has no clock and
   no timeout. `pending_len` is how a supervisor notices a link that was
   sent half a packet and will never be sent the rest.
8. **Every LE event arrives inside the LE Meta Event, code 0x3E.** The
   subevent code is `params[0]` (Core Vol 4 Part E section 7.7.65). This
   package does not split it out, because doing so means knowing every
   LE event's shape.
9. **A Command Complete event names the command it answers.**
   `command_complete_opcode` reads the opcode from the parameters and
   `command_complete_status` reads the status byte after it (section
   7.7.14). Both answer `None` for an event that is not one or is too
   short.
10. **`decode` refuses synchronous and isochronous data.** Both are
    legal H4 type bytes, so the deframer will hand them to you, and
    `decode` answers `UnsupportedPacketType` naming the type rather than
    pretending the packet was malformed.
11. **An ACL packet's `broadcast` field is always 0 on LE.** It is a
    field rather than a constant so that a decode reports what the
    controller actually sent (Core Vol 4 Part E section 5.4.2).
12. **Every multi-byte field is little-endian.** The opcode, the ACL
    handle and flags word, and the ACL payload length are all written
    low byte first.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device
with no heap allocator, and the compiler checks that claim on every
build. Here the claim covers the whole package. Every function is
integer arithmetic over bytes the caller supplies, and the deframer's
state is a value the caller owns rather than a buffer this package
hides. `feed_byte` is the entry point on that path, because a byte
arrives in a register and building a list to pass it would allocate.

```bash
novo build --target=nrf52-qemu src/main.nv
```

`tests/embedded_probe.nv` is that claim as a program that either builds
or does not. **It does not build today.** The probe's own `main` builds
a list literal to make a packet, and a list literal is a heap allocation
the embedded tier refuses (`E4000`). The signatures the probe exercises
are unchanged, and a caller that supplies bytes it already holds meets
no such refusal.

## What is not included

- **Synchronous and isochronous data.** Voice over SCO and LE Audio over
  ISO have their own packet shapes. The type bytes are recognised so a
  deframer can say what it saw, and `decode` refuses them by name.
- **A transport other than H4.** The three-wire UART transport, USB and
  SDIO each frame packets differently. H4 is the one this package
  writes, and it is the one nearly every LE controller offers.
- **What any packet means.** Nothing here knows that opcode 0x0C03 is a
  reset or that event 0x3E subevent 0x01 is a connection. A host stack
  owns that knowledge, and a codec that had it would have to be
  rewritten for every new command.
- **Flow control.** The controller's buffer count arrives in the Number
  Of Completed Packets event, and counting credits against it is the
  host stack's work. This package decodes the event and keeps no state
  about it.
- **A working `drain`.** `drain` reads a `ByteSource` until it has
  nothing and returns every packet. It declares whatever effects the
  source declares, so it costs `[io]` against a host's serial port and
  nothing against a buffer of captured bytes. It cannot be called
  today. A call from another module specialises it, and the
  specialised copy keeps the `[e]` in its effect clause while losing the
  bound that bound it (`effect error: 'e' is not an effect` [E3005]).
  Since a package's public function is by construction called from
  another module, write the `feed` and `take` loop yourself until that
  is fixed. The signature is published unchanged.
- **A radio, a controller and a UART.** Bytes arrive as bytes and leave
  as bytes. Opening the port is the caller's work.

## Related packages

- [l2cap-nv](https://novo-lang.org/packages/l2cap-nv) is the layer
  above. It turns the payload of an ACL data packet into channels, and
  it depends on this package so that its fragmentation and reassembly
  speak this package's `AclData` and `Boundary` types.
- [att-nv](https://novo-lang.org/packages/att-nv) and
  [smp-nv](https://novo-lang.org/packages/smp-nv) are two layers above,
  on the L2CAP channels 0x0004 and 0x0006. Neither talks to HCI
  directly.
- [ble-link-codec-nv](https://novo-lang.org/packages/ble-link-codec-nv)
  is the layer below, for a program that drives a radio itself rather
  than a controller. HCI is what you use when the radio is behind
  firmware; the link-layer codec is what you use when it is not.
- `std.net` in the standard library opens sockets. It has nothing to do
  with Bluetooth, and this package needs none of it.

## Tests

```bash
novo test tests/hci_codec_tests.nv     # 17 tests against the signatures
```

The byte strings the suite asserts against are the Bluetooth Core
Specification's own, Volume 4 Parts A and E: the section 2 H4 framing of
HCI_Reset, the section 5.4.1 command header, the section 5.4.2 ACL
header and the section 7.7.14 Command Complete event. The suite checks
that an opcode packs and unpacks, that a type byte and its layer round
trip, that a scan of a prefix asks for more and a scan of a bad type
byte asks to resynchronise, that each packet survives encoding and
decoding, and that a second packet fed in one chunk stays buffered until
it is taken.

`drain` has no test. Calling it from a test module does not compile, for
the defect named under "What is not included".

The tests compile today and fail at run, each on the `not implemented:
hci_codec.<fn>` panic that is its body. That is the expected state of an
interface release. They turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| The `OGF_*` and `EVT_*` constants | yes (they are constants) |
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

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
