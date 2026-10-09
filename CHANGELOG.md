# Changelog

## Unreleased

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
