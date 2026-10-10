# TCI-X: TCI with extensions for full radio control

*Draft 0.1. First written 27 Sep 2026, published 7 Oct 2026. Editor: Mike G0KAD, with the KADRAD project.*

*Base: Expert Electronics TCI v2.0 (12 Jan 2024), published at [github.com/ExpertSDR3/TCI](https://github.com/ExpertSDR3/TCI) under an MIT permission notice, which allows this derived specification. The notice is reproduced in Appendix B. TCI-X is an independent extension. It is not made or endorsed by Expert Electronics.*

---

## 1. Purpose

TCI v2.0 is a good transport. It's one WebSocket carrying text commands and binary audio, with the server pushing every state change to all clients. But it was written around one SDR program, and it leaves three gaps when it's used with conventional CAT/CI-V radios:

1. **Ambiguity.** Some behaviour is unspecified (command forms, echo rules, what a refusal looks like, sensor units when the radio can't measure something). Servers differ, so clients have to guess. Section 3 pins these down.
2. **Missing functions.** There are no memories, preamp/attenuator, named filters, RF or mic gain, compressor, VOX, break-in, repeater duplex and tone, extra meters or VFO select, A=B and swap. Section 5 adds them.
3. **No discovery.** A client can't find out which commands a radio really supports, or their ranges. Section 4 adds a capability manifest, so a client can build a panel without per-radio knowledge.

**Design rules:**

- **A strict superset.** An unmodified TCI v2.0 client works against a TCI-X server. A TCI-X client works against a plain TCI server, using the base set only.
- **TCI syntax throughout.** Extensions use TCI's `name:arg,arg;` form, so any TCI parser can read them. There's no JSON and no second channel.
- **Engineering units on the wire.** Hz, dBm, dB, W, V, A, ratios and percentages. Converting raw radio values (Icom `0000`–`0255`, BCD and the like) is the server's job, never the client's.
- **Written to be adopted.** Extension names follow TCI's style and carry no vendor prefix (RFC 6648 explains why prefixes hurt adoption). If Expert Electronics or others want to fold any of this into a future TCI, the names can stay as they are.

## 2. Conventions

- "Server" means a TCI-X server, such as `kadrad-hub`, the KADRAD project's reference server. "Base" means TCI v2.0 as published.
- **Must**, **should** and **may** are used in the sense of RFC 2119.
- `trx` is the transceiver index and `ch` the channel index (0 = A, 1 = B), as in the base spec.
- **String arguments in extension commands are percent-encoded** for `%`, `:`, `,`, `;` and control characters (e.g. `name:%2C` for a comma). Base commands are unchanged; `cw_macros` text still follows the base rules.
- Booleans are `true`/`false`. Numbers use `.` as the decimal point, with no thousands separator.

## 3. Base TCI v2.0, clarified

A TCI-X server must behave as below for **base** commands. These rules come from bench work with real radios and existing TCI servers, where each of these points was found to vary or to be unclear.

| # | Topic | Rule |
|---|---|---|
| B1 | Case | Commands are case-insensitive (base §3.1). The server always replies in lower case. |
| B2 | Echo | Every accepted set is echoed to **all** clients with the value actually applied (clamped or rounded to what the radio did). A client reconciles its optimistic UI against the echo. |
| B3 | Refusal | A set the radio can't perform is answered with an echo of the **current** value (e.g. `xit_enable:0,false;`). A TCI-X client also receives `error:` (§5.1). A read of an unsupported command gets `error:` for TCI-X clients and silence for base clients. |
| B4 | Audio format forms | `audio_samplerate`, `audio_stream_sample_type`, `audio_stream_channels` and `audio_stream_samples` are accepted both in the base form (`audio_samplerate:12000;`) and trx-indexed (`audio_samplerate:0,12000;`). The reply uses the form received. |
| B5 | Audio format scope | Audio format is **per connection**. One client choosing 12 kHz int16 mono doesn't change another client's 48 kHz float32 stream. |
| B6 | Block length | The server sends RX blocks of exactly `audio_stream_samples` frames. Clients should still accept any length, because base servers vary. |
| B7 | S-meter | If the radio has an S-meter, the server sends `rx_channel_sensors` in calibrated dBm while `rx_sensors_enable` is on, at the requested interval. It also sends the deprecated `rx_sensors` for older clients. |
| B8 | TX sensors | While transmitting, the server sends `tx_sensors` at the requested interval. Values the radio can't measure are sent as `0`, and the manifest (§4) marks them unavailable. Power is in watts, derived from the radio's meter and its rated power for the band. |
| B9a | TX frequency | `tx_frequency` is where the radio really transmits: VFO B in split, else VFO A moved by any duplex offset. |
| B9 | Channel B | On a radio with two VFOs and one receiver, `ch 1` is VFO B. `vfo:0,1,<hz>` sets VFO B without swapping when the radio can; otherwise it is refused (B3). |
| B10 | Drive | `drive` is a percentage of the rated power **for the current band**. The manifest gives the rated watts per band, so a client can label the knob in watts. |
| B11 | Modulations | `modulations_list` names every mode the radio has, in lower case: base names (`lsb`, `usb`, `cw`, `am`, `nfm`, `wfm`, `digl`, `digu`, …) plus the TCI-X names in §5.9. Base clients ignore names they don't know. |
| B12 | CW | `cw_macros`, `cw_macros_stop`, `cw_macros_speed` and `cw_keyer_speed` must work on any radio that can send CW from text or from a keyer. If the radio can't, they're refused (B3) and absent from the manifest. |
| B13 | Several clients | Several clients may connect at once, as the base spec intends. All receive every state push. The takeover model (a new client evicts the old one) is **not** used. |
| B14 | TX ownership | The client that keys (`trx:…,true` or `tune:…,true`) owns TX until it unkeys or disconnects. While it owns TX, other clients' key requests are refused. Only the owner's `TX_AUDIO_STREAM` is transmitted. |
| B15 | TX safety | The server unkeys when the TX owner's connection drops, when its TX audio stops for longer than `tx_stream_audio_buffering` + 1 s while keyed with source `tci`, and at the TX timeout (§5.8). The unkey is pushed to all clients as `trx:…,false`. |
| B16 | Refused base commands | Commands with no meaning for the radio (`dds`, `if`, `iq_*`, `spot*` on a radio with no scope, for example) are refused per B3 rather than dropped silently. Where the radio has a scope (`cap:spectrum`), the server keeps the spots itself and pushes them to TCI-X clients ([drafts/spectrum.md](drafts/spectrum.md) §4). |
| B17 | Passband edges | `rx_filter_band:<trx>,<low>,<high>` gives the passband's edges in Hz from the carrier, `low < high`. Upper-sideband modes (USB, DIGU, RTTY) use positive edges, and lower-sideband modes (LSB, DIGL, RTTY-R) negative ones (LSB 300–2700 Hz is `-2700,-300`). In CW the edges are relative to the CW pitch tone, 0 being centred on it, and CW-R mirrors them. AM and FM are symmetric about 0. The server pushes the line whenever the width, centre or mode changes. A radio whose filters are only named presets doesn't offer it (use `filter`). |
| B18 | Implied TX source | A base client (one that hasn't sent `tcix:`) that has started RX audio and keys with no source (`trx:0,true;`, as WSJT-X does) is keyed as if it had sent source `tci`: the server requests its TX audio with `TX_CHRONO`. A TCI-X client must name the source; with none, the radio uses its own audio input. |
| B19 | Tune | `tune:<trx>,<bool>` keys the radio's own tune carrier, for an antenna tuner or an amplifier's tuner to tune on. It's keying: the server applies `trx`'s checks, and B14 and B15 apply (no TX audio is involved). `tune_drive:<trx>,<pct>` is the carrier's power, a percentage of the rated power for the current band as `drive` (B10), kept apart from `drive`. While the carrier is on, the server pushes `tune:<trx>,true` and `trx:<trx>,true`. A server whose radio has no tune carrier leaves both out of the manifest and refuses them (B3). |

## 4. Negotiation and the capability manifest

### 4.1 Opt-in

On connect, the server sends the base init burst, ending with `ready;`, followed by the state push, exactly as in the base spec. The `protocol:` line keeps the base form (`protocol:kadrad-hub,2.0;`) so base clients parse it as before.

A TCI-X client sends, at any time after connecting:

```
tcix:1.0;
```

A TCI-X server replies on that connection only:

```
tcix:1.0;
cap:…;          (one line per capability, §4.2)
cap_end;
<current state of every extension command in the manifest>
```

A base server ignores `tcix:` (base §3.1: invalid commands are ignored). If there's no reply within 2 s of `ready;`, the client stays in base mode.

Until a connection opts in, the server sends it **only** base commands, so base clients never see extensions.

### 4.2 `cap` lines

```
cap:<command>,<access>,<type>[,<params>…];
```

- `access`: `r`, `w` or `rw`.
- `type` and params:
  - `bool`
  - `range,<min>,<max>,<step>,<unit>`: e.g. `cap:drive,rw,range,0,100,1,pct;`
  - `enum,<v1>,<v2>,…`: e.g. `cap:agc_mode,rw,enum,fast,mid,slow,off;`
  - `action`: a command with no value (e.g. `vfo_swap`).
  - `meter,<unit>,<min>,<max>[,<red_from>]`: for `meter` names (§5.6), e.g. `cap:meter.alc,r,meter,pct,0,100,100;`
  - `info,<value>…`: fixed facts, e.g. `cap:rated_power,r,info,hf,100,6m,100,2m,50,70cm,35;`
- `cap:support,r,info,<level>;` says how far the server's support for this radio has been checked, so a client can say so: `rigctld` (basic control through Hamlib's `rigctld`, as good as Hamlib's description of the radio), `documentation` (written from the manufacturer's reference, not yet tried on a radio; a server should keep these receive-only), `probed` (checked against a real radio by the probe kit) or `bench-verified` (tested on a radio, transmit included).
- Base commands the server supports are listed too, so the manifest is the complete list. A command that isn't in the manifest isn't supported.
- A `cap` line can arrive again later (e.g. `rf_gain` becomes unavailable in a mode). The newest line wins, and `cap:<command>,none;` removes one. When a command comes back (the K3's APF on a change to CW), the server sends its `cap` line and then its current value.

## 5. Extension commands

All are bidirectional unless marked. The server echoes and pushes them just like base commands (B2), but only to TCI-X clients.

### 5.1 Session

| Command | Form | Notes |
|---|---|---|
| `tcix` | `tcix:<version>;` | Opt-in (§4.1). |
| `error` | `error:<command>,<code>[,<text>];` | Server → client. Codes: `unsupported`, `range`, `busy` (another client owns TX), `tx_active` (not allowed while transmitting), `duplex` (key refused: a duplex offset is set outside FM, so the radio would transmit off the dial frequency), `tx_lock` (key or change refused: the TX frequency or power would fall outside the server's configured TX lock, a site limit such as one dummy load's frequency and power), `tx_audio` (key with `tci` audio refused: the radio's audio input wouldn't take the client's audio over this link), `tx_timer` (key refused: over a network link that can drop while keyed, the radio's own TX timer is off), `tx_where` (key refused: the radio reports a TX frequency other than the server's), `radio` (the radio rejected or didn't answer), `device` (an amplifier or rotator, or the program that owns it, refused or can't be reached; the text says why, e.g. someone else is in control of it, §5.13–5.14), `timeout`. |
| `radio_state` | `radio_state:<state>;` | Server → client: `connected`, `no_response`, `powered_off`, `port_lost`. |
| `power` | `power:<bool>;` | Radio power on/off, where the radio supports it. |
| `dial_lock` | `dial_lock:<bool>;` | The radio's own dial lock: its front-panel tuning is locked. Tuning over TCI still works, but a client with a dial of its own should stop it too while the lock is on, so the lock means the same everywhere. One for the radio, so no trx index. |
| `audio_codec` | `audio_codec:<trx>,pcm\|opus;` | RX audio coding for this connection (B5), offered as `cap:audio_codec,rw,enum,pcm,opus`. Both forms, as B4. `opus` takes `audio_samplerate` (8/12/24/48 kHz) but ignores `audio_stream_channels`, `audio_stream_sample_type` and `audio_stream_samples`: blocks are mono, one packet each (§6). Default `pcm`. TX audio stays PCM. |

### 5.2 VFO

| Command | Form | Notes |
|---|---|---|
| `vfo_select` | `vfo_select:<trx>,<ch>;` | Which VFO is active (A = 0, B = 1). |
| `vfo_equal` | `vfo_equal:<trx>;` | Action: A=B. |
| `vfo_swap` | `vfo_swap:<trx>;` | Action: A↔B. |
| `tx_watch` | `tx_watch:<trx>,<bool>;` | Listen on the transmit frequency while it differs from the receive one (split): Icom XFC, Yaesu TXW. |
| `band` | `band:<trx>,<name>;` | The radio's band key, using the radio's own band stack: the server saves where the radio is into the current band's latest stack entry, then applies the named band's latest entry (frequency, mode, filter, DATA; duplex and tone in FM only). The radio's stack stays the one record, so its own band key agrees. Write-only; the server echoes `band:<trx>,<name>` and pushes the new `vfo`/`modulation`. Names come from `cap:band,w,enum,160m,…`; a band the list lacks (the IC-7100 has no 60 m stack) is the client's to handle. |

### 5.3 Receiver front end

| Command | Form | Notes |
|---|---|---|
| `preamp` | `preamp:<trx>,<value>;` | Enum from the manifest, e.g. `off,p1,p2`. |
| `attenuator` | `attenuator:<trx>,<db>;` | Enum of dB steps, e.g. `0,12`. |
| `rf_gain` | `rf_gain:<trx>,<pct>;` | 0–100. |
| `af_gain` | `af_gain:<trx>,<pct>;` | 0–100. The radio's own speaker and headphone level (the K3's AF GAIN), not the audio stream. |
| `rx_audio_level` | `rx_audio_level:<trx>,<pct>;` | 0–100. The level of the RX audio stream itself (§6) as the server sends it, for every client: what WSJT-X's own input attenuator does, done once at the source. A server offers it where the radio sets that level apart from its speaker (the FLEX-6000's DAX RX gain). |
| `squelch` | `squelch:<trx>,<pct>;` | Squelch knob position, 0–100. The base `sql_level` is a threshold in dB, which knob-style radios can't honour. |
| `agc_mode` | *(base)* | Base command. Its values come from the manifest enum (the base spec lists `normal,fast,off`; radios have more). |
| `agc_time` | `agc_time:<trx>,<value>;` | The decay time of the AGC speed in use (`agc_mode`), from the manifest's enum: `off`, or seconds (`0.1`, `0.2`, …). The list can follow the mode (Icom: 0.1–6.0 s in SSB, CW and RTTY, 0.3–8.0 s in AM), and the server re-sends the cap line when it changes. Each AGC speed keeps its own time, so the server pushes `agc_time` again after `agc_mode` changes. |
| `rx_nb_level`, `rx_nr_level` | `rx_nr_level:<trx>,<pct>;` | Levels alongside the base `rx_nb_enable` / `rx_nr_enable`. |
| `antenna` | `antenna:<trx>,<name>;` | Enum from the manifest. |
| `rx_antenna` | `rx_antenna:<trx>,<bool>;` | Receive on a separate receive-only input (the K3's RX ANT, with the KXV3) instead of `antenna`. Transmit stays on `antenna`. Withdrawn (`cap:rx_antenna,none`) if the radio reports it hasn't got one. |

### 5.4 Filters

| Command | Form | Notes |
|---|---|---|
| `filter` | `filter:<trx>,<ch>,<name>;` | Named preset (`fil1`, `fil2`, `fil3`, or the radio's own names). The base `rx_filter_band` stays for radios with a continuously adjustable passband, and is pushed after any preset change so base clients see the edges. |
| `pbt` | `pbt:<trx>,<inner_hz>,<outer_hz>;` | Twin passband tuning. |
| `rx_xfil` | `rx_xfil:<trx>,<n>;` | Read-only: the crystal (roofing) filter the radio has chosen for the current passband, numbered as on the radio (the K3's FL1–FL5). `cap:rx_xfil,r,enum,1,2,3,4,5`. |
| `rx_filter_center` | `rx_filter_center:<trx>;` | Action: put the passband back at the mode's normal centre, keeping its width (the K3's SHIFT/NORM). The server then pushes `rx_filter_band`. The manifest lists it as `cap:rx_filter_center,w,action`, and lists `rx_filter_band` as `cap:rx_filter_band,rw,range,<min_width>,<max_width>,<step>,hz`, a range that applies to the width (`high - low`). |
| `notch` | `notch:<trx>,<bool>,<hz>;` | Manual notch: on/off, and the audio frequency it cuts (what you hear), within `cap:notch,rw,range,<lo>,<hi>,1,hz`. The range can depend on the mode (the IC-7100: −1040 to 4040 Hz in SSB, the CW pitch ± 2540 Hz, −5060 to 5100 Hz in AM), and the server re-sends the cap line when it changes. |
| `notch_width` | `notch_width:<trx>,<name>;` | The manual notch's width, from the manifest's enum (Icom: `wide`, `mid`, `narrow`). |
| `apf` | `apf:<trx>,<bool>;` | Audio peaking filter (CW). |
| `filter_shape` | `filter_shape:<trx>,<shape>;` | The DSP filter's skirt, from the manifest's enum (Icom: `sharp`, `soft`). |
| `twin_peak` | `twin_peak:<trx>,<bool>;` | RTTY twin peak filter (Icom TPF). A radio may refuse it unless its RTTY tones are the standard ones (Icom: mark 2125 Hz, shift 170 Hz), with `error:twin_peak,radio`. |

### 5.5 Transmitter

| Command | Form | Notes |
|---|---|---|
| `mic_gain` | `mic_gain:<trx>,<pct>;` | |
| `comp` | `comp:<trx>,<bool>,<pct>;` | Speech compressor on/off and level. |
| `monitor` | `monitor:<trx>,<bool>,<pct>;` | The radio's transmit monitor (hearing your own signal) on/off and level. Base TCI's `mon_enable` and `mon_volume` are the SDR program's own monitor, in dB; this is the radio's. |
| `vox` | `vox:<trx>,<bool>;` | |
| `vox_gain` | `vox_gain:<trx>,<pct>;` | VOX sensitivity, 0–100. |
| `anti_vox` | `anti_vox:<trx>,<pct>;` | Anti-VOX level (how much receive audio it ignores), 0–100. |
| `break_in` | `break_in:<trx>,<mode>;` | `off`, `semi` or `full`. |
| `break_in_delay` | `break_in_delay:<trx>,<tenths>;` | Semi break-in's hang time in tenths of a dot at the current speed (Icom 2.0–13.0 dots: `20`–`130`), as `cap:break_in_delay,rw,range,20,130,1,ddot`. |
| `cw_pitch` | `cw_pitch:<trx>,<hz>;` | |
| `essb` | `essb:<trx>,<bool>;` | Extended SSB: a wider TX bandwidth (the K3's 3.0–4.0 kHz). |
| `tx_bandwidth` | `tx_bandwidth:<trx>,<name>;` | The SSB transmit bandwidth preset, from the manifest's enum (Icom TBW: `wide`, `mid`, `narrow`). |
| `atu` | `atu:<trx>,<bool>;` | The antenna tuner in or bypassed (Icom `1C 01`, with an external tuner such as the AH-705). |
| `atu_tune` | `atu_tune:<trx>;` | Action: start a tune. **It transmits**: the server applies the same checks as keying (`trx`): another client owning TX is `busy`, an unknown TX frequency is `radio`, a TX lock or a receive-only radio refuses it. The radio keys itself and drops back when the tuner is done; `trx` pushes show it, and the server's TX timeout applies. |
| `data_mode` | `data_mode:<trx>,<bool>[,<filter>];` | For radios where DATA is a flag on top of the mode (Icom `1A 06`). `digu`/`digl` in `modulation` set it too. |
| `cw_memory` | `cw_memory:<trx>,<slot>[,<text>];` | The radio's CW keyer memories (Icom `1A 02`, M1–M8). With no text, a read; with text, a write, and an empty text clears the slot. The server answers with the slot as the radio holds it. `text` is percent-encoded. The manifest gives the slots and the length: `cap:cw_memory,rw,info,8,70;`. Characters the keyer can't hold are refused (`range`). Storing never transmits; a client sends a memory's text with `cw_macros`. |

### 5.6 Meters

```
meter:<trx>,<name>,<value>;          server → client
meters_enable:<bool>[,<interval_ms>];  client → server
```

Names: `alc` (%), `comp` (dB), `vd` (V), `id` (A), `po` (W), `swr` (ratio), `s` (dBm), `temp` (°C), and an external amplifier's `amp_…` meters (§5.13). The manifest's `cap:meter.<name>` line gives the unit, the scale and the red zone. This is a generic channel, so new meters don't need new commands. `s`, `po` and `swr` are **also** sent in the base `rx_channel_sensors` / `tx_sensors` (B7, B8) for base clients.

### 5.7 FM and repeaters

| Command | Form | Notes |
|---|---|---|
| `duplex` | `duplex:<trx>,<dir>,<offset_hz>;` | `dir`: `simplex`, `minus` or `plus`. |
| `tone` | `tone:<trx>,<mode>,<hz>;` | `mode`: `off`, `tone` (CTCSS encode), `tsql` or `dtcs`. For `dtcs`, the value is the three-digit code (`023`), not Hz; its polarity stays the radio's. The reply carries the frequency of the active mode (the encode tone when `off`). The tones the radio accepts are listed in `cap:tone_hz,r,info,67.0,69.3,…;`, and its DTCS codes in `cap:dtcs_codes,r,info,023,025,…;`. |

### 5.8 Safety

| Command | Form | Notes |
|---|---|---|
| `tx_timeout` | `tx_timeout:<seconds>;` | The server's TX time limit. It can only be lowered from the server's configured maximum. `0` means the server maximum. |

### 5.9 Modes

TCI-X adds these `modulation` names where the radio has them: `cwr`, `rtty`, `rttyr`, `dv` (D-STAR: selecting the mode only, TCI-X 1.0 carries no D-STAR data), `digfm` (FM with DATA), `psk`, `pskr`. Base names keep their meanings.

### 5.10 Memories

| Command | Form | Notes |
|---|---|---|
| `memory_list` | `memory_list:<trx>[,refresh];` → one `memory` line per channel, then `memory_list_end:<trx>;` | Read the channel table. The server may send lines as it reads the radio, and cache them. `refresh` reads the radio again. |
| `memory` | `memory:<trx>,<channel>,<name>,<rx_hz>,<mode>,<duplex_dir>,<offset_hz>,<tone_mode>,<tone_hz>;` | One channel. `channel` is an opaque id from the server (`2-01` on Icom: group 2, channel 1). `name` is percent-encoded. Empty channels are omitted. A write stores the channel when the manifest has `cap:memory,w,action`; the server answers with the channel as the radio then holds it. |
| `memory_recall` | `memory_recall:<trx>,<channel>;` | Recall into the VFO: the radio stays in VFO mode with the channel's frequency, mode, duplex and tone. The server pushes the resulting `vfo`, `modulation`, `duplex` and `tone`, then echoes `memory_recall`. |
| `memory_mode` | `memory_mode:<trx>,<bool>[,<channel>];` | The radio's own memory mode (Icom V/M): on, on a channel, or back to VFO mode. Memories are selected, never written. On radios that can't report V/M (IC-7100), the server reports what it last set. While it's on, the server refuses to retune the VFO (`error:vfo,range`). |
| `memory_clear` | `memory_clear:<trx>,<channel>;` | Empties a channel, where the manifest has `cap:memory_clear,w,action`. Echoed when done. |

### 5.11 Audio shaping

Settings the radio keeps per mode, or per preset, in its menus: the operator sets each one up rather than working it, so a client edits any set, not only the one in use. The set names are the radio's, listed in a `.sets` info line. Which set applies in which mode is the radio's own rule (Icom: `ssb` covers LSB, USB and their DATA forms).

| Command | Form | Notes |
|---|---|---|
| `eq` | `eq:<trx>,<path>,<set>,<source>,<v1>,…,<vn>;` | Receive (`rx`) or transmit (`tx`) equaliser or tone controls, as in [drafts/eq.md](drafts/eq.md) §3: `cap:eq.<path>,<access>,eq,<min>,<max>,<step>,<unit>,<band>,…;` and `cap:eq.<path>.sets,r,info,<set>,…;`. Icom tone controls: `cap:eq.rx,rw,eq,-5,5,1,step,bass,treble;`. `source` is `radio`, `sent` or `unknown`, and empty in a client's write. |
| `rx_audio_filter` | `rx_audio_filter:<trx>,<set>,<low_hz>,<high_hz>;` | The receive audio high-pass (`low_hz`) and low-pass (`high_hz`) edges, `0` for an edge that is off (Icom: "through"). `cap:rx_audio_filter,rw,edges,hz;`, with the values each edge can take in `cap:rx_audio_filter.low,r,info,…;` and `cap:rx_audio_filter.high,r,info,…;`, and the sets in `cap:rx_audio_filter.sets,r,info,…;`. `low_hz` must be below `high_hz` unless either is `0`. |
| `tx_bandwidth_edges` | `tx_bandwidth_edges:<trx>,<preset>,<low_hz>,<high_hz>;` | The edges of each `tx_bandwidth` preset (§5.5), plus `data` where the radio has a separate SSB-DATA bandwidth. Caps as `rx_audio_filter`: `edges`, then `.low`, `.high` and `.sets` (the presets). |

### 5.12 Scan

The radio's own scans. Radios can't report whether one is running, so these are write-only (`w`), and a client shouldn't show a scan as running from its own request alone: the frequency moving is what shows it.

| Command | Form | Notes |
|---|---|---|
| `scan` | `scan:<trx>,<kind>;` | Start a scan, or `stop`. Kinds from `cap:scan,w,enum,stop,…`: `programmed` (between the radio's programmed edges), `delta_f` (around the frequency, ± `scan_span`), `fine_programmed`, `fine_delta_f` (slowing on a signal), `memory`, `select_memory`, `mode_select` (memories in the current mode). The radio may refuse one it can't run now (no edges set, not in memory mode) with `error:scan,radio`. |
| `scan_span` | `scan_span:<trx>,<khz>;` | The ΔF scan's half-width, from `cap:scan_span,w,enum,5,10,20,50,100,500,1000`. |
| `scan_resume` | `scan_resume:<trx>,<bool>;` | Whether a scan resumes after stopping on a signal. |

### 5.13 External amplifier

A linear amplifier in the station's transmit path, which the server reads from the amplifier's own controller (a serial port, or a program that owns that port). The radio's `po` and `swr` meters show what goes into the amplifier. These show what goes to the antenna. Read-only except Operate/Standby (`amp_operate`), its tuner (`amp_tune`) and its antenna (`amp_antenna_next`): everything else is done from the amplifier's own controls or its own program.

A server that's set up with an amplifier announces it in the manifest, and keeps the announcement while the amplifier isn't answering. `amp_state` says whether the readings are live.

```
cap:amp,r,info,<maker>,<model>,<rated_w>;        e.g. cap:amp,r,info,SPE,Expert 1.3K-FA,1300;
```

| Command | Form | Notes |
|---|---|---|
| `amp_state` | `amp_state:<state>;` | Server → client: `connected` (readings are live), `no_response` (the amplifier, or the program that owns its port, is reachable but the amplifier isn't answering, e.g. it's switched off), `unreachable` (the server can't reach the amplifier's controller). While it isn't `connected`, a client should show the amplifier's readings as unavailable, not as zero. |
| `amp_operate` | `amp_operate:<bool>;` | `true` in Operate (the amplifier amplifies), `false` in Standby (RF passes through). A client may write it where the manifest says `rw`: the server asks the amplifier's controller to switch, and the push of the new state is the answer. Refused with `device` while the amplifier isn't `connected`, or while its controller won't take the change (someone else is in control of it there), and with `tx_active` while the amplifier transmits. |
| `amp_tx` | `amp_tx:<bool>;` | Read-only. The amplifier reports it is transmitting. |
| `amp_power_level` | `amp_power_level:<level>;` | Read-only. The amplifier's power setting, from `cap:amp_power_level,r,enum,…` (SPE: `low`, `mid`, `high`). |
| `amp_band` | `amp_band:<name>;` | Read-only. The band the amplifier is set to, with names as in `band` (§5.2). A client may warn when it differs from the radio's. |
| `amp_input` | `amp_input:<n>;` | Read-only. Which of the amplifier's inputs (radios) is selected, numbered as on the amplifier. |
| `amp_antenna` | `amp_antenna:<n>,<atu>;` | Read-only. The amplifier's transmit antenna, numbered as on the amplifier, and its tuner: `on`, `bypass` or `none`. |
| `amp_antenna_next` | `amp_antenna_next;` | Action, where the manifest has `cap:amp_antenna_next,w,action`: the amplifier's next antenna, as its own antenna key steps (the SPE steps through the antennas set up for the band). The push of `amp_antenna` is the answer. Refused with `tx_active` while the amplifier transmits, and with `device` as `amp_operate`. |
| `amp_tune` | `amp_tune;` | Action, where the manifest has `cap:amp_tune,w,action`: starts the amplifier's own tuner (its tune key). It doesn't transmit by itself: the amplifier tunes on the RF it's given, which a client can send with `tune` (B19) at the power the amplifier asks for. Refused with `device` as `amp_operate`. |
| `amp_warning` | `amp_warning:<text>;` | Read-only. The amplifier's current warning in its own words (percent-encoded), or empty when there's none. A warning doesn't stop the amplifier (SPE: `ATU BYPASSED`, `OVERHEATING`). |
| `amp_alarm` | `amp_alarm:<text>;` | Read-only. The amplifier's current alarm, or empty when there's none. An alarm means the amplifier has protected itself (SPE: `SWR EXCEEDING LIMITS`, `INPUT OVERDRIVING`). |

The amplifier's readings are meters (§5.6) on the trx it belongs to, named `amp_<name>`, each with its `cap:meter.amp_<name>` line:

| Meter | Unit | What it reads |
|---|---|---|
| `amp_po` | W | Output power. |
| `amp_swr` | ratio | The SWR the amplifier sees (after its tuner, if any). |
| `amp_swr_ant` | ratio | The antenna's SWR before the tuner, where the amplifier measures it. |
| `amp_vd` | V | PA supply voltage. |
| `amp_id` | A | PA current. |
| `amp_temp` | °C | The hottest PA temperature sensor. |

A server may add other `amp_<name>` meters (SPE: `amp_temp_combiner`); the manifest describes each one. Amplifier meters are TCI-X only: the base `tx_sensors` keep the radio's own readings.

The manifest lists every command above, so a client can gate on it (§4.2). For an SPE Expert 1.3K-FA:

```
cap:amp,r,info,SPE,Expert 1.3K-FA,1300;
cap:amp_state,r,enum,connected,no_response,unreachable;
cap:amp_operate,rw,bool;
cap:amp_tx,r,bool;
cap:amp_power_level,r,enum,low,mid,high;
cap:amp_band,r,enum,160m,80m,60m,40m,30m,20m,17m,15m,12m,10m,6m,4m;
cap:amp_input,r,enum,1,2;
cap:amp_antenna,r,enum,1,2,3,4;
cap:amp_antenna_next,w,action;
cap:amp_tune,w,action;
cap:amp_warning,r,info;
cap:amp_alarm,r,info;
cap:meter.amp_po,r,meter,w,0,1300;
cap:meter.amp_swr,r,meter,ratio,1,5,3;
cap:meter.amp_swr_ant,r,meter,ratio,1,5,3;
cap:meter.amp_vd,r,meter,v,0,60;
cap:meter.amp_id,r,meter,a,0,40;
cap:meter.amp_temp,r,meter,c,0,100;
cap:meter.amp_temp_combiner,r,meter,c,0,100;
```

`amp_antenna`'s list gives the antenna numbers; its tuner value is always one of `on`, `bypass` and `none`. `amp_warning` and `amp_alarm` carry text, so their lines list no values. Temperatures use the unit `c` (°C). After `cap_end`, the state push includes `amp_state` and every amplifier value known so far.

### 5.14 Antenna rotator

An antenna rotator, which the server reads and turns through the rotator's own controller (a serial port, or a program that owns that port). Bearings are whole degrees clockwise from north, 0–360.

A server that's set up with a rotator announces it in the manifest, and keeps the announcement while the rotator isn't answering:

```
cap:rot,r,info,<maker>,<model>;                    e.g. cap:rot,r,info,Idiom Press,Rotor-EZ;
cap:rot_state,r,enum,connected,no_response,unreachable;
cap:rot_heading,r,range,0,360,1,deg;
cap:rot_target,r,info;
cap:rot_turn,w,range,0,360,1,deg;
```

| Command | Form | Notes |
|---|---|---|
| `rot_state` | `rot_state:<state>;` | Server → client, as `amp_state` (§5.13): `connected` (readings are live), `no_response` (the rotator's controller is reachable but the rotator isn't answering), `unreachable`. While it isn't `connected`, a client should show the heading as unavailable. |
| `rot_heading` | `rot_heading:<deg>;` | Read-only. Where the antenna points, pushed when it changes. |
| `rot_target` | `rot_target:<deg>;` | Read-only. Where the antenna is turning to, or empty when it isn't turning. It clears when the antenna gets there (whoever asked for the turn: a client, the controller's own page, or another program). |
| `rot_turn` | `rot_turn:<deg>;` | Write-only. Turn the antenna to a bearing; `rot_target` and then `rot_heading` pushes are the answer. Refused with `range` for a bearing outside 0–360, and with `device` while the rotator isn't `connected` or its controller won't take the turn (someone else is in control of it there). |

A rotator is TCI-X only, and one per server: no trx index.

## 6. Binary streams

These are unchanged from the base spec, except for RX audio after `audio_codec:0,opus` (§5.1). Each RX_AUDIO block then carries one Opus packet (RFC 6716) as its payload, with `codec` = 1 (0 is PCM, as in base TCI), `channels` = 1, `sample_rate` the rate it decodes to, and `length` the samples it decodes to, as for PCM. The packet's size is the payload's: the message length less the 64-byte header. The reference server sends 40 ms packets at 24 kbit/s (voice, up to 8 kHz wide): about 40 kbit/s with the headers, against 0.2 Mbit/s for 12 kHz int16. Base clients never see Opus: they can't ask for it.

## 7. Versioning

The TCI-X version (`tcix:1.0`) is independent of the base TCI version. Minor versions only add commands. A major version may change existing ones. A client should send the highest version it knows, and the server replies with the version it will speak.

The document version and the wire version are separate. Drafts 0.x of this document describe wire version `tcix:1.0`. Until the document reaches 1.0, wire 1.0 may still change, and changes are listed in [CHANGELOG.md](CHANGELOG.md). From then on, wire 1.0 is fixed and only grows through 1.1, 1.2, and so on.

## 8. Open points

- **Name.** TCI is Expert Electronics' protocol. We're asking them how they'd like a derived work to refer to it, and we'll follow their wishes. On the wire the opt-in token is `tcix` (no hyphen), matching the lower-case, underscore style of TCI command names.
- **Collisions with future base commands.** If a later TCI adds a command with one of our names but different arguments, the base meaning wins, and TCI-X renames its own in a major version.
- **Spectrum scope data** (for radios that output it: the Flex, IC-7300, IC-705) is left out of 1.0. Draft, not in 1.0: a per-connection binary stream for panadapter clients, plus spots kept by the server. See [drafts/spectrum.md](drafts/spectrum.md).
- **RX and TX equalisers** (draft, not in 1.0): one `eq` command for graphic EQs and tone controls, with write-only EQs (the K3's TX EQ) marked as such. See [drafts/eq.md](drafts/eq.md).
- **Amplifier interlocks.** A server refusing a key-up while the amplifier is in alarm (a new `error` code), and one action that starts the amplifier's tuner and sends the carrier until it's done, are left for later.
- **Authentication** for servers reachable beyond localhost. TCI has none. Options include a token in the WebSocket URL or relying on a VPN.

## Appendix A. IC-7100 mapping (informative)

How the reference server maps TCI-X onto the first radio it supported. It's an example for server authors, not part of the protocol.

| TCI-X / base | CI-V |
|---|---|
| `vfo` ch 0 / ch 1 | `05` / `25 01` (E4+) |
| `modulation`, `data_mode` | `06`, `1A 06` |
| `filter` | `06` data byte, `26` |
| `drive` | `14 0A` (percentage of 255) |
| `rf_gain`, `sql_level` | `14 02`, `14 03` |
| `mic_gain`, `comp` | `14 0B`, `14 0E` + `16 44` |
| `cw_keyer_speed` | `14 0C` (6–48 WPM over 0–255) |
| `cw_macros` | `17` (up to 30 characters per frame; break-in on) |
| `cw_pitch` | `14 09` |
| `preamp`, `attenuator` | `16 02`, `11` |
| `agc_mode` | `16 12` (`fast`, `mid`, `slow`) |
| `rx_nb_enable`, `rx_nr_enable`, `rx_anf_enable` | `16 22`, `16 40`, `16 41` |
| `vox`, `break_in` | `16 46`, `16 47` |
| `rit_enable`, `rit_offset` | `21 01`, `21 00` |
| `split_enable` | `0F` |
| `duplex`, `tone` | `0F 11/12` + `0D`, `16 42/43` + `1B 00/01` |
| `vfo_select`, `vfo_equal`, `vfo_swap` | `07 00/01`, `07 A0`, `07 B0` |
| `memory*` | `08`, `09`, `0A`, `0B`, `1A 00` |
| `trx`, `tune` | `1C 00`, `1C 01` (external AH-4 only) |
| `power` | `18 00` / `18 01` |
| `rx_channel_sensors`, `meter` | `15 02` (S), `15 11` (Po), `15 12` (SWR), `15 13` (ALC), `15 14/15/16` (COMP, Vd, Id) |

The meter calibration (raw `0000`–`0255` to dBm, W, ratio and so on) is the server's job (§1, engineering units), and is checked against the radio. The table is from Icom's CI-V reference for firmware E6.

## Appendix B. Base specification notice

TCI Protocol v2.0: Copyright (c) 2023 "Expert Group" LLC, Taganrog.

Permission is hereby granted, free of charge, to any person obtaining a copy of this software (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions: The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
