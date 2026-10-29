# TCI-X draft: a second receiver

*Draft, not part of TCI-X 1.0. First written 29 Oct 2026. The reference server implements it for an Airspy HF+ beside an Icom IC-7100, as the radio's sub receiver; the KADRAD client shows it as a SUB strip with its own tuning knob and a balance between the two receivers' audio. Comments are welcome before this goes into a minor version.*

## 1. Why

A second receiver is what sets top radios apart: listening to the DX and to the pileup at once, or watching another spot in the band. A budget radio can have one from a cheap SDR that the server runs beside it, on the radio's IF or on an antenna (with a switch that takes it off while the radio transmits). Base TCI numbers transceivers (`trx`), so the second receiver is `trx 1`. What's needed is a way to say what it offers, which may be much less than the radio does (no transmitter, its own modes and tuning range), without confusing clients that know only one transceiver.

## 2. On the wire

- **TCI-X clients only.** A base client is told `trx_count:1` and sees nothing of trx 1, as before. A client that opts in (§4.1) is told of it in the manifest.
- **Announced** by `cap:trx1,r,info,<facts>` before `cap_end`. The facts so far: `receive_only` (it never transmits). A client that doesn't know this draft ignores the line.
- **What it offers** as `cap:trx1.<command>,…` lines: the same form as trx 0's (§4.2), for the commands trx 1 takes. A command it doesn't list, it doesn't take, whatever trx 0 does. Per-connection commands (the audio stream format, `rx_sensors_enable`, `meters_enable`) aren't listed: they apply to both.
- **Its limits** come from its own lines: `cap:trx1.vfo,rw,range,<min>,<max>,1,hz` and `cap:trx1.modulation,rw,enum,…` stand for `vfo_limits` and `modulations_list`, which have no trx index.
- **Its state** follows the manifest as ordinary trx-indexed lines with index 1 (`vfo:1,0,7074000`, `modulation:1,usb`, `rx_filter_band:1,100,3200`, `agc_mode:1,mid`), and changes are pushed as for trx 0.
- **Commands** for it are the ordinary ones with index 1. A command it doesn't list is refused (`error:<command>,unsupported`); `trx:1,true` is never taken.
- **Meters:** its S-meter as `meter:1,s,<dBm>` (with `cap:trx1.meter.s`) and `rx_channel_sensors:1,0,<dBm>`, at the connection's intervals.
- **Audio:** `audio_start:1` and `audio_stop:1` start and stop its stream, apart from trx 0's. Its blocks are RX_AUDIO with `receiver` = 1, in the connection's format (§6).

## 3. Where it can listen

A second receiver that shares the radio's view (an SDR on the radio's IF, or one on an antenna that follows the radio's dial for the panadapter) can only hear what's in that view. The server takes any frequency in `cap:trx1.vfo`'s range; while it's out of view, trx 1 is silent and sends no S-meter reading. A later revision may announce the reach (`cap:trx1.reach,r,info,<hz>`) so a client can say so.

## 4. Example

```
cap:trx1,r,info,receive_only;
cap:trx1.vfo,rw,range,500,260000000,1,hz;
cap:trx1.modulation,rw,enum,lsb,usb,digl,digu,cw,am;
cap:trx1.rx_filter_band,rw,range,50,9000,50,hz;
cap:trx1.agc_mode,rw,enum,fast,mid,slow;
cap:trx1.meter.s,r,meter,dbm,-127,-13;
cap:trx1.audio_start,w,action;
cap:trx1.audio_stop,w,action;
cap_end;
…
vfo:1,0,7074000;
modulation:1,usb;
```

## 5. Open points

- A second receiver of the radio's own (an IC-7610's sub, an FTDX101's): the same lines would serve, with `cap:trx1` saying what it shares with trx 0 (the antenna, the band).
- Stereo: playing the two receivers in the two ears is the client's choice; the server sends each as mono or as the connection's format asks.
- The reach (§3) and a `receive_only` trx 1 being taken off its antenna while trx 0 transmits (`rx_enable:1,false` pushed while it's off?).
