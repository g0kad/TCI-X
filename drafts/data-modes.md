# TCI-X draft: data modes decoded by the server

*Draft, not in TCI-X 1.0. Comments welcome.*

## 1. Why the server decodes

A server beside the radio can decode FT8, FT4 and similar modes from the radio's receive audio itself, and send clients the decodes as text. Then:

- a client needs no audio stream at all, which matters on a mobile link (a decode is a few dozen bytes; audio is 40 kbit/s or more, and a dropout loses decodes);
- a client needs no decoder, so a phone or a browser can work data modes;
- the server, which keys the radio, also times transmissions against its own clock.

A client that wants to decode for itself still can, from the audio stream, as today.

## 2. Manifest

A server that decodes announces the modes, and the state of its clock (the modes depend on UTC to about a second):

```
cap:data,rw,enum,FT8,FT4;                 the modes this server decodes
cap:data_enable,rw,bool;
cap:data_partner,rw,info;
cap:data_my_call,rw,info;
cap:data_clock,r,enum,synced,unsynced,unknown;
```

These are TCI-X only. `data`, `data_enable`, `data_partner` and `data_my_call` take a `trx` index; each receiver decodes on its own.

## 3. Commands

| Command | Form | Notes |
|---|---|---|
| `data` | `data:<trx>,<mode>;` | The mode this receiver decodes, from the cap line, or empty for none. Shared by every client of the radio (decoding costs the server's processor), and set by the operator (§4.4) like tuning. A listen-only radio may decode: decoding never transmits. The server pushes it on change. Decoding follows the audio the radio gives, whatever its mode and frequency: a client that wants FT8 on 14.074 MHz also sets the frequency and mode. |
| `data_enable` | `data_enable:<trx>,<bool>;` | Per connection, as `rx_sensors_enable`: whether this client wants `data_decode` lines. Off by default. |
| `data_decode` | `data_decode:<trx>,<mode>,<period>,<stage>,<snr>,<dt>,<hz>,<ap>,<text>;` | Server → client: one decode. `period` is the period's start in Unix milliseconds (UTC). `stage` is `early` (partway through the period, from the start of the message), `partner` (the station in `data_partner`, as soon as its signal ends) or `final` (after the period). A later stage never repeats an earlier stage's decode. `snr` dB in 2500 Hz, `dt` seconds from the mode's nominal start, `hz` the audio frequency of the signal's lowest tone. `ap` is empty, or the a priori assumption that helped decode it, in WSJT-X's marks (`a1` CQ, `a2` my call, `a3` my call and the partner, `a9` a reply expected from the previous periods). `text` is the message as WSJT-X shows it, percent-encoded (a DXpedition message contains `;`). |
| `data_waterfall` | `data_waterfall:<trx>,<ms>,<levels>;` | Server → client, while a mode is set, about four a second: one line of the waterfall. `ms` is the time of its first sample in Unix milliseconds (UTC). `levels` is one hex digit per 6.25 Hz column from 200 Hz to 3000 Hz (448 digits), `0` at the column's noise and `f` strongest. Announced by `cap:data_waterfall,r,info;`. |
| `data_period` | `data_period:<trx>,<mode>,<period>,<stage>,<count>;` | Server → client, after each stage of each period, even with no decodes: the stage is complete, with `count` decodes. A client can clear its "decoding" indicator, and knows a quiet band from a stalled decoder. |
| `data_partner` | `data_partner:<trx>,<call>,<hz>,<dt>;` | The station being worked, as the operator chose it, and where it was last heard: the server decodes it first at the end of each period (`partner`), and assumes its call (`a3`). Empty clears it. Set by the operator; pushed on change. |
| `data_my_call` | `data_my_call:<trx>,<call>;` | The call the station signs. The server assumes it in messages to this station (`a2`) only once the station has transmitted (a CQ, or a call), when someone may be answering: assumed otherwise, it found only false decodes. With `data_partner`, it's assumed with the partner's call (`a3`). Set by the operator; pushed on change. |
| `data_clock` | `data_clock:<state>,<offset_ms>;` | Server → client, on change and every minute: whether the server's clock is synchronised (`synced`, `unsynced`, `unknown`), and its offset from UTC where known (empty where not). A client shows it beside the decodes, and warns when it isn't `synced`. |

Errors: `range` for a mode not in the cap line; `control` or `access` as for other operator settings (§4.4, §4.3).

## 4. What the server keeps

- **Hashed calls.** A message can carry a call as a hash (`<...>`). The server keeps a table of calls it has decoded in full and shows a hashed call in full once it knows it, for every client alike.
- **The last periods**, for `a9`: having decoded `A B …`, the server looks for `B A …` in the next period at the frequency A last used.

## 5. Transmitting

A server that can transmit a data mode adds to the manifest:

```
cap:data_tx,rw,info;
cap:data_tx_stop,w,bool;
cap:data_tune,w,bool;
cap:data_tx_level,rw,range,-40,0,1,db;
```

| Command | Form | Notes |
|---|---|---|
| `data_tx` | `data_tx:<trx>,<slot>,<hz>,<text>;` | Client → server: send `text` (percent-encoded, as in `data_decode`) once, in the mode set by `data`, at audio frequency `hz` (200 to 3000), in the next period of `slot`: `0` for even periods, `1` for odd, counting whole periods since the Unix epoch (for FT8, `0` starts at :00 and :30). A period that began less than two fifteenths of its length ago still counts: the signal then starts late. A new `data_tx` replaces one that is waiting. One sent while transmitting takes the following period. |
| `data_tx_stop` | `data_tx_stop:<trx>;` | Client → server: cancel what's waiting and unkey now, if transmitting. Any client may stop it. |
| `data_tx_level` | `data_tx_level:<trx>,<db>;` | The level of the generated audio into the radio, in dB of full scale, as in the range on its cap line. Set by the operator, shared by every client and pushed on change. It applies at once, during a transmission too, so the operator can bring the radio's ALC down while watching it. |
| `data_tune` | `data_tune:<trx>,<bool>,<hz>;` | A steady tone at audio frequency `hz` at the `data_tx_level` level, starting now, for tuning an antenna. The server ends it after at most 15 s, or on `false`. It takes every check a transmission does except the clock's, and shows in `data_tx_state` with the text `TUNE`. |
| `data_tx_state` | `data_tx_state:<trx>,<state>,<period>,<hz>,<text>;` | Server → client, on change: `idle` (nothing waiting or going, after a message finished), `queued` (waiting for `period`), `sending`, or `stopped` (ended before it finished: `data_tx_stop`, a refusal, the radio unkeyed). The last three are empty when `idle` or `stopped`. |

Errors: `range` for a message the mode can't carry, or a bad slot or `hz`; `state` when no mode is set or the radio isn't in an upper-sideband mode; `clock` when `data_clock` isn't `synced`; and `access`, `control`, `busy`, `interlock` as for `trx`.

The transmission belongs to the connection that asked for it: if that connection drops, the server unkeys and forgets what was waiting. Its rules:

- The server generates the audio and keys the radio at the period's start, with every check `trx` has (§3 B14, B15, `interlock`, `access`, `control`), and refuses when `data_clock` isn't `synced`.
- **Never unattended.** A transmission follows from an operator's choice: answering a CQ or calling CQ starts from a client request, and the server may carry a QSO it started on (report, RR73, 73) without asking again. A server offers no mode that calls or answers stations by itself with nobody there.
- A message that answers two stations at once (the DXpedition type, `A RR73; B <C> -12`) concerns each recipient only for its own half.

## 6. Not covered

Logging the QSOs that result is a separate draft. Continuous modes (RTTY, PSK31) need a text stream rather than per-period decodes, and will get their own commands.
