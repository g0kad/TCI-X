# TCI-X

**TCI with extensions for full radio control.** It's an open, backwards-compatible superset of Expert Electronics' [TCI v2.0](https://github.com/ExpertSDR3/TCI), aimed at conventional CAT/CI-V transceivers as well as SDRs.

**Status: Draft 0.1** (7 Oct 2026). It's implemented and in daily use, but it may still change. Comments and proposals are welcome.

## Why

TCI is a good transport: one WebSocket carrying text commands and binary audio, with every state change pushed to every client. WSJT-X, JTDX, loggers and contest programs already speak it. But it was written around one SDR program, and with conventional radios it has three gaps:

1. **Ambiguity.** Servers differ on command forms, echoes, refusals and sensor units, so clients have to guess. TCI-X pins these down (§3).
2. **Missing functions.** TCI has no memories, preamp or attenuator, named filters, RF or mic gain, compressor, VOX, break-in, repeater duplex and tone, extra meters, or VFO select, A=B and swap. TCI-X adds them (§5).
3. **No discovery.** A client can't find out what a radio really supports. TCI-X adds a **capability manifest** (§4), so a client can build its controls from what the radio has, without knowing about each radio in advance.

## Design rules

- **A strict superset.** An unmodified TCI client works against a TCI-X server, and a TCI-X client works against a plain TCI server, using the base set only.
- **Opt-in.** A client sends `tcix:1.0;`. Until it does, the server sends it base TCI only.
- **TCI syntax throughout:** `name:arg,arg;`, with no JSON and no second channel.
- **Engineering units on the wire** (Hz, dBm, W, ratio, %). Raw radio values are the server's problem.
- **No vendor prefixes**, so any of it can be folded into a future TCI as is.

## Documents

| | |
|---|---|
| [TCI-X.md](TCI-X.md) | The specification: base clarifications, negotiation and the manifest, extension commands, the Opus RX audio codec. |
| [drafts/spectrum.md](drafts/spectrum.md) | Draft: panadapter spectrum stream and server-kept spots. |
| [drafts/eq.md](drafts/eq.md) | Draft: RX and TX equalisers. |
| [examples/](examples/) | Real init bursts and manifests from the reference server, for five radios. |
| [CHANGELOG.md](CHANGELOG.md) | Changes between drafts. |

## Implementations

- **kadrad-rig** (server, reference implementation): drives the radio directly over CAT, CI-V or the SmartSDR API, and serves it as TCI-X. It supports the Icom IC-7100 and IC-705, the Elecraft K3, the QRP Labs QMX+, the Yaesu FTDX10, FlexRadio 6000/8000, and radios through Hamlib's `rigctld`. Part of the KADRAD project.
- **KADRAD** (client): a TCI/TCI-X radio front panel that builds its controls from the manifest.

Plain TCI clients such as WSJT-X work unchanged against kadrad-rig.

If you implement TCI-X, in a server or a client, please open an issue so it can be listed here.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). In short: open an issue first for anything that adds or changes a command, and describe the radio behaviour behind it.

## Relationship to TCI

TCI is Expert Electronics' protocol, published under an MIT permission notice. TCI-X is an independent extension. It is not made or endorsed by Expert Electronics. Where a future TCI version defines a command with one of our names, the base meaning wins (TCI-X §8).

## Licence

MIT. See [LICENSE](LICENSE). TCI-X is derived from the TCI Protocol v2.0, Copyright (c) 2023 "Expert Group" LLC, Taganrog, which is published under the same MIT notice. That notice is reproduced in [TCI-X.md, Appendix B](TCI-X.md#appendix-b-base-specification-notice).
