# Changelog

## Unreleased

- §5.2: `vfo_link:<trx>,<bool>`, VFO B following VFO A. Added for the Elecraft K3 (`LN`).
- §5.10: `cap:memory_list.read,r,info,on_request` for a radio that can only recall a memory to read it: the server reads the channels only on `memory_list:<trx>,refresh` and puts the radio back after; a plain `memory_list` returns what it has. `memory_recall` then uses the radio's own recall, before the channel is listed if need be, and there's no `memory_mode`. Added for the Elecraft K3 (`MC`).
- New §4.4, control: on a radio with logins, one operator at a time may change the radio while everyone else watches (`cap:control`, `control:<user>,<since>`, `control_take`, `control_release`, `control_request` and `control_answer`, `control_asked`, `control_refused`), with the new `error` code `control`. Control is taken by the first change, never changes hands while transmitting, and is taken back after an idle time, closing the operator's connections; unkeying is never refused. Added for kadrad-hub at a club station shared by its members.
- §5.13: `amp_power_level_next` (the amplifier's next power setting, as its power key) and `amp_drive_down` / `amp_drive_up` (a step of the radio's drive, for an amplifier that sets it in Operate), all actions. Added for an SPE Expert 1.3K-FA driving a FLEX-6400 (its POWER and − / + keys).
- §4: a server with no receive audio for a radio leaves `audio_start`, `audio_stop` and `audio_codec` out of the manifest. Added for kadrad-hub radios set up for control only.
- §5.3: `rx_nr_lms_enable`/`rx_nr_lms_level`, `rx_anf_lms_enable`/`rx_anf_lms_level` and `rx_wnb_enable`/`rx_wnb_level`: an adaptive (LMS) noise reduction and notch alongside the base ones, and a wideband noise blanker, each with a level. Where the radio doesn't report one it's announced `w` and echoed as set. Added for the FLEX-6000 on firmware 4 (`lms_nr`, `lms_anf`, `wnb`; a FLEX-6400 on a basic licence takes these, and refuses the subscription filters `nrf` and `anft`).
- §5.13: `amp_power`, the amplifier switched on or off where the server can (a relay on its remote on line), announced as `cap:amp_power,rw,bool`, with or without `cap:amp`. §5.1: `power` may be switched by the server itself (a relay on the radio's remote power line), refused with `access` or `device`. Added for kadrad-hub with kadrad-power at the club (a FlexRadio and an SPE on a relay board).
- New §4.3, login: a server may require one on a radio (`login_required:<nonce>`, `login:<user>,<proof>`, `login:ok,<role>`), with an HMAC-SHA256 proof over a scrypt key so the password never crosses the link, and the roles `listen`, `operate`, `station` and `admin`. `error:login,access,…` for a failed login. The open point on authentication becomes one on encryption. Added for kadrad-hub's per-radio logins.
- `rx_antenna_port` (§5.3): the receive input chosen apart from `antenna`, which is then the transmit antenna. Added for FlexRadio's `rxant` and `txant`.
- New `error` code `access` (§5.1): refused by the server's rules for who may do what, such as a radio set to listen only (its clients may tune and listen, never transmit); the text says why. Such a radio's `receive_only` is `true`, so clients hide their transmit controls. Added for kadrad-hub's per-radio access rules.
- New `error` code `interlock` (§5.1): a key-up or tune refused because a device in the radio's transmit path that it shares with another radio on the same server (an amplifier, an antenna), or a rule that the two don't transmit together, is in use by that radio. The text names both. Added for kadrad-hub serving several radios that share an amplifier or antennas.
- `rx_audio_level` (§5.3): the level of the RX audio stream as the server sends it, apart from `af_gain` (the radio's speaker). Added for the FLEX-6000, whose DAX RX audio arrives near full scale with AGC off (its DAX RX gain).
- B19: base TCI's `tune` and `tune_drive` pinned down. `tune` keys the radio's tune carrier under `trx`'s checks and ownership; `tune_drive` is its power, a percentage of rated as `drive`, kept apart from it. Added for the FLEX-6000 (`transmit tune`, `tunepower`).
- §5.13: `amp_tune` (start the amplifier's tuner) and `amp_antenna_next` (its next antenna), both actions. Added for an SPE Expert 1.3K-FA (its TUNE and ANTENNA keys).
- New §5.14, antenna rotator: `cap:rot`, `rot_state`, `rot_heading`, `rot_target` and `rot_turn`. Added for an Idiom Press Rotor-EZ, read and turned through its own control server.
- §5.13: `amp_operate` may be written (Operate/Standby), where the manifest says `rw`. New `error` code `device` (§5.1): an amplifier or rotator, or its controller, refused or can't be reached.
- §5.13: the manifest lines for the amplifier commands (`cap:amp_state`, `amp_operate`, `amp_tx`, `amp_band`, `amp_input`, `amp_antenna`, `amp_warning`, `amp_alarm`, and the `amp_…` meters), with the unit `c` for temperatures. Before, only `cap:amp` and `cap:amp_power_level` were named.
- The reference server is renamed from kadrad-rig to **kadrad-hub** (KADRAD, 9 Oct 2026). Its `protocol:` line is now `protocol:kadrad-hub,2.0;`; nothing else on the wire changes, and clients shouldn't rely on the server name.

- New §5.13, external amplifier (read-only): `cap:amp`, `amp_state`, `amp_operate`, `amp_tx`, `amp_power_level`, `amp_band`, `amp_input`, `amp_antenna`, `amp_warning`, `amp_alarm`, and `amp_…` meters (§5.6). Added for an SPE Expert 1.3K-FA, read through its own control server.

- `cw_memory` (§5.5): the radio's CW keyer memories, read and stored. Added for the IC-705 (`1A 02`, M1–M8).
- `memory` writes and `memory_clear` (§5.10): announced by `cap:memory,w,action` and `cap:memory_clear,w,action`; a write is answered with the channel as stored. Added for the IC-705 (`1A 00`).
- `error` codes (§5.1): `tx_audio`, `tx_timer` and `tx_where`, so a client can say why a key-up was refused. Added for the IC-705 (MOD input `1A 05 0118`/`0119`, Time-Out Timer over WLAN, TX frequency `1C 03`).
- `tone` (§5.7): `dtcs` takes the three-digit code, with the codes in `cap:dtcs_codes`. Built for the IC-705 (`16 4B`, `1B 02`).
- `atu` and `atu_tune` (§5.5): the antenna tuner, and a tune that transmits under the same checks as keying. Added for the IC-705 with an AH-705 (`1C 01`).
- `tx_watch` (§5.2), and a new §5.12 Scan: `scan`, `scan_span`, `scan_resume`, write-only. Added for the IC-705 (`1C 02`, `0E`).
- `dial_lock` (§5.1), `agc_time` (§5.3) and `twin_peak` (§5.4). Added for the IC-705 (`16 50`, `1A 04`, `16 4F`).
- New §5.11, audio shaping: `eq` (from drafts/eq.md), `rx_audio_filter` and `tx_bandwidth_edges`, settings kept per mode or preset. Added for the IC-705 (`1A 05`: RX HPF/LPF, RX and TX tone, TBW edges).
- `monitor:<trx>,<bool>,<pct>` (§5.5): the radio's transmit monitor, on/off and level. Added for the IC-7100 (`16 45`, `14 15`).
- `notch` (§5.4): `<hz>` is the audio frequency cut, and the range may follow the mode. New `notch_width` (§5.4). Added for the IC-7100 (`16 48`, `14 0D`, `16 57`).
- `vox_gain`, `anti_vox`, `break_in_delay` and `tx_bandwidth` (§5.5), and `filter_shape` (§5.4). Added for the IC-7100 (`14 16`, `14 17`, `14 0F`, `16 58`, `16 56`).

## Draft 0.1 (7 Oct 2026)

First public draft, describing wire version `tcix:1.0` as implemented by kadrad-rig.

- Base TCI v2.0 clarifications B1–B18: echoes, refusals, audio format forms and scope, sensors, TX frequency, TX ownership and safety, passband edges, implied TX source.
- Negotiation (`tcix:`) and the capability manifest (`cap:`, `cap_end`), including `cap:support` levels.
- Extension commands: session (`error`, `radio_state`, `power`, `audio_codec`), VFO (`vfo_select`, `vfo_equal`, `vfo_swap`, `band`), receiver front end, filters, transmitter, meters, FM and repeaters, safety (`tx_timeout`), extra modes, memories.
- Opus-coded RX audio (`audio_codec:<trx>,opus`).
- Drafts, not in 1.0: spectrum stream and spots, equalisers.
