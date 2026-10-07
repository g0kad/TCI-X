# TCI-X draft: spectrum (panadapter) stream

*Draft, not part of TCI-X 1.0. First written 6 Oct 2026. The reference server implements it for FlexRadio 6000/8000 (`spectrum_enable`, `spectrum_span`, `spectrum_center`, `spectrum_edges`, the `0x100` frames, and `cap:spectrum_edges,r,info`), and for the Icom scope radios from Icom's CI-V reference (not yet tried on a radio). `spectrum_mode` and `spectrum_ref` are not built yet. Comments are welcome before this goes into a minor version.*

## 1. What radios offer

| | FlexRadio 6000/8000 | Icom IC-705 / IC-7300 / IC-7610 |
|---|---|---|
| **Data** | FFT rows as VITA-49 on UDP (class `0x8003`); waterfall tiles separately (`0x8004`) | Scope waveform over CI-V (`27 00`), once output is turned on (`27 11`); about 475 points per sweep, split over several frames |
| **Points and rate** | Set by the client on the radio (`xpixels`, `fps`) | Fixed by the radio; the rate is limited by the link (USB CI-V) |
| **Span** | Any bandwidth, centred anywhere | Centre mode (fixed spans, ±2.5 kHz to ±500 kHz) or fixed mode (edges per band) |
| **Level** | Calibrated, against the pan's own `min_dbm`/`max_dbm`; about 0.4 dB resolution at 256 rows | 0–160, relative; reference level adjustable |
| **Measured** (FLEX-6400) | 500 points at 15/s: 0.09 Mbit/s; 1000 at 17/s: 0.28; 2000 at 27/s: 0.89 | — |

Radios that send no scope data over CAT (IC-7100, K3, QMX+, FTDX10, among others) get no `cap:spectrum` line, so a client offers no panadapter for them.

## 2. Principles

- **Per connection, opt-in.** As with the audio format (TCI-X B5), each client asks for its own spectrum stream, at its own point count and rate. A client that never enables it costs nothing.
- **The client sets the cost.** Points × rate × bytes per point is the bandwidth. 1,000 points at 15 rows a second, one byte each, is about 120 kbit/s plus headers. That suits a VPN or 4G. A desktop on a LAN can ask for more.
- **The server reduces, keeping peaks.** When the radio gives more points than the client asked for, the server combines neighbouring points by taking the maximum, so a narrow CW signal can't fall between points. Where the radio can do the work itself (the Flex's `xpixels` and `fps`), the server asks the radio for the client's numbers.
- **Span is the radio's setting.** Centre, span and reference level are radio state, shared by every client and pushed like any other change. Point count and rate belong to each connection.
- **Honest levels.** The manifest says whether values are calibrated dBm or relative dB.

## 3. Proposed commands

### 3.1 Manifest

```
cap:spectrum,rw,info,<unit>,<max_points>,<max_rate>;
cap:spectrum_span,rw,enum,<span_hz>,<span_hz>,…;        (radios with fixed spans)
cap:spectrum_span,rw,range,<min_hz>,<max_hz>,<step_hz>;  (radios with any span)
cap:spectrum_center,rw,range,<lo>,<hi>,1,hz;             (only where the view can move without retuning)
cap:spectrum_mode,rw,enum,center,fixed;                  (only where the radio has both)
cap:spectrum_ref,rw,range,<min_db>,<max_db>,<step_db>;   (only where adjustable)
cap:spectrum_edges,r,info;
```

`<unit>` is `dbm` (calibrated) or `db` (relative). The Flex sends `dbm`, the Icoms `db`.

### 3.2 Commands

| Command | Form | Notes |
|---|---|---|
| `spectrum_enable` | `spectrum_enable:<trx>,<bool>[,<points>,<rate>];` | Per connection. Starts or stops this client's stream. Points and rate are clamped to the manifest's maximums, and the echo gives the values in use. |
| `spectrum_span` | `spectrum_span:<trx>,<hz>;` | Radio state. In centre mode the span is centred on the VFO. |
| `spectrum_mode` | `spectrum_mode:<trx>,center\|fixed;` | Radio state, where the radio has both. |
| `spectrum_center` | `spectrum_center:<trx>,<hz>;` | Radio state: moves the view without retuning (a drag on the panadapter). Pushed with `spectrum_edges`. |
| `spectrum_edges` | `spectrum_edges:<trx>,<low_hz>,<high_hz>;` | Pushed whenever the displayed range changes (tuning in centre mode, a span change). Writable in fixed mode only. |
| `spectrum_ref` | `spectrum_ref:<trx>,<db>;` | Radio state, where adjustable. |

Tuning from the panadapter is the ordinary `vfo` command; nothing new is needed.

### 3.3 Binary frames

A new stream type in the base TCI stream header, from a range reserved for TCI-X (`type` = `0x100`, so a future base type can't collide). One frame is one row:

| Field | Value |
|---|---|
| `receiver` | trx |
| `sample_rate` | 0 |
| `format` | 0: one unsigned byte per point |
| `codec`, `crc` | 0 |
| `length` | number of points |
| `type` | `0x100` (spectrum) |
| `channels` | 1 |
| `reserv[0]`, `reserv[1]` | frequency of the first point, Hz (uint64, low word first) |
| `reserv[2]` | span, Hz (the last point is at start + span) |
| `reserv[3]` | level of byte value 0, in 0.01 dB (int32; dBm or dB per the manifest) |
| `reserv[4]` | step per byte value, in 0.01 dB (for example 50 = 0.5 dB, giving a 127.5 dB range) |
| `reserv[5]` | flags: bit 0 = row taken while transmitting |
| `data` | the points, lowest frequency first |

The client draws both the spectrum trace and the waterfall from these rows.

## 4. Spots

Base TCI already has spots: a client sends `spot:<call>,<mode>,<hz>,<argb>,<text>;`, `spot_delete:<call>;` and `spot_clear;`, and the server sends `rx_clicked_on_spot:<trx>,<ch>,<call>,<hz>;` when one is clicked. ExpertSDR draws them on its own panorama.

A TCI-X server with no panorama of its own keeps the spot list instead: by callsign, at most 500, each for an hour. It pushes it to TCI-X clients as `spot`, `spot_delete` and `spot_clear` (the whole list when a client opts in). A panadapter client sends `rx_clicked_on_spot:<trx>,<ch>,<call>,<hz>` when a spot is clicked, and the server passes it to every client, with the legacy `clicked_on_spot:<call>,<hz>` too.

So a DX cluster client (or any TCI logger) adds spots, a panadapter client draws them, and a click on one tunes the radio and tells every client which spot it was. The clients never talk to each other directly.

## 5. Open points

- **Passband fallback.** For radios with no scope data, a server could compute a narrow spectrum from RX audio (the passband only) and serve it the same way.
- **Several pans.** The Flex can run more than one panadapter. This draft gives one per trx; a second would want a pan index.
- **Compression.** Rows change slowly; delta or run-length coding could halve the bandwidth again. Not before it's measured.
