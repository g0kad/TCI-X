# Examples

What the reference server (kadrad-hub) sends a client that connects and then opts in with `tcix:1.0;`. The files are the server's golden test copies: each is regenerated from the server's radio profile and checked in its test suite, so they show exactly what the server sends.

Each file has two parts, split by a `---` line:

1. **The base init burst**, which every client gets: `protocol` … `ready`, then `start` and the state push. A plain TCI client sees only this.
2. **The reply to `tcix:1.0;`**: the `cap:` manifest, `cap_end`, and the current value of each extension command.

The `;` terminators are left off, one command per line. Radio state (frequencies, modes) is the server's default before it has read the radio.

| File | Radio | Notes |
|---|---|---|
| [ic7100_tcix.txt](ic7100_tcix.txt) | Icom IC-7100 | CI-V. The fullest manifest: memories, duplex and tone, all meters. |
| [k3_tcix.txt](k3_tcix.txt) | Elecraft K3 | ASCII CAT. |
| [qmx_tcix.txt](qmx_tcix.txt) | QRP Labs QMX+ | A small QRP radio: a short manifest. |
| [ftdx10_tcix.txt](ftdx10_tcix.txt) | Yaesu FTDX10 | `cap:support,r,info,documentation`: written from the manual, receive-only. |
| [flex_tcix.txt](flex_tcix.txt) | FlexRadio 6000/8000 | Over IP (the SmartSDR API). The radio describes itself, so there's no model file. Shown with transmit not enabled (`receive_only:true`). |
| [amp_tcix.txt](amp_tcix.txt) | SPE Expert 1.3K-FA (amplifier) | Not a radio: the lines a server adds for an external amplifier (§5.13). Its two parts are the `cap:` lines, which go before `cap_end`, and the values in the state push after it, here with the amplifier answering, in Operate and transmitting. |

The `cap` lines are the best guide to how a server describes a radio. For example, the attenuator is `0,12` dB on the IC-7100, `0,10` on the K3 and `0,6,12,18` on the FTDX10, and the QMX+ has none, so it has no line at all. The `cap:rated_power` lines differ the same way.
