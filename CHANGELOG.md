# Changelog

## Unreleased

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
