# TCI-X draft: several slices in one radio

*Draft, not part of TCI-X 1.0. First written 5 Nov 2026, from [#4](https://github.com/g0kad/TCI-X/issues/4). Comments are welcome before this goes into a minor version.*

## 1. Why

**One radio, one transmitter, several receivers.** An SDR can run several receivers at once in the same hardware: a FLEX-6400 has two slices, a FLEX-6600 four, and a Hermes Lite 2 or ANAN can cut up to eight receivers from its hardware. Each can be tuned anywhere the radio receives, with its own mode and audio, and any one of them can be the one the transmitter uses. This draft is about those **slices**.

It isn't about several radios. A server that serves several radios gives each its own connection (its own port, its own `trx 0`), as the reference server does; nothing here changes that.

**Slices are not separate radios.** A server shouldn't present one radio's slices as several radios, each on its own connection with its own `trx 0`: they share one transmitter, one power amplifier and one set of transmit settings, which separate radios don't. Shown as separate radios, each appears to have a transmitter of its own, so TX ownership (B14) no longer covers them and two clients could key the same transmitter from two "radios"; a change to a setting they share (`pa_enable`, `predistortion`, `band_filter`, and usually `drive`) changes all the others without their clients being told; and a client can't show the slices together. A server that wants a connection per slice for programs that know only `trx 0` uses slice ports (§5), which are the same radio.

Base TCI numbers these slices as transceivers (`trx`), as ExpertSDR does, and some base TCI clients choose one by its index. The TCI fork of fldigi lets the user pick a "receiver slice" from 1 to 8 (trx 0 to 7), and WSJT-X offers "TCI Client RX1" and "RX2" (trx 0 and 1). On ExpertSDR each index is a slice with its own frequency, mode and audio, and keying it transmits from it. A server for an SDR with several slices can offer the same, so that these clients work unchanged.

## 2. On the wire

- **Base clients too.** A server whose radio has N slices says `trx_count:N` to every client, base or TCI-X, and they take indices 0 to N−1.
- **A slice** takes the same commands as trx 0, with its own index: it tunes (`vfo`, `dds`, `if`), has its own mode, filter and the rest of trx 0's receive settings, and its own audio (`audio_start:<trx>`, RX_AUDIO blocks with `receiver` = the index).
- **One transmitter.** Keying index K (`trx:<K>,true`, `tune:<K>,true`) makes slice K the one the transmitter uses, if it isn't already (a FlexRadio's TX slice), and transmits on its frequency, with its mode and transmit settings. B14 applies across all the slices: while one client owns TX, a key-up on any index from another client is refused with `busy`. A slice that can't transmit says so (§3).
- **Shared and lasting.** A slice is the radio's state, not the connection's. Two clients choosing the same index share that slice, as two clients share trx 0 today, and it stays as it was left after its last client goes. (The exception is a port where each connection has a transceiver of its own, §6.)
- **What TCI-X clients learn from the manifest.** A slice that can do all trx 0 does needs no `cap:trx<K>` lines: with none, index K takes everything trx 0 offers. A server lists `cap:trx<K>.<command>` lines only where an index differs, and then they say exactly what it takes, as in drafts/second-receiver.md §2.

## 3. Slices that differ

- **Receive only:** `cap:trx<K>,r,info,receive_only`. It never transmits: `trx:<K>,true` and `tune:<K>,true` are refused with `unsupported`.
- **Fewer commands:** `cap:trx<K>.<command>` lines, as in drafts/second-receiver.md, list what that index takes.

## 4. With a second receiver

A receive-only second receiver for TCI-X clients only (drafts/second-receiver.md), such as a separate SDR the server runs beside the radio, takes the first index after the radio's slices: index N. Base clients aren't told of it, so they still see `trx_count:N`. With a radio of one slice that's index 1, as before.

## 5. Slice ports

A program that knows only `trx 0` (one WSJT-X per band, watching several bands at once) can't choose a slice by index. For it a server may also offer a **slice port** for each slice: another listening port where `trx 0` means slice K. A slice port is a view of the same radio, not another radio:

- **The same slice.** What a client does on slice K's port is done to slice K, and clients of the main port see it as index K, and the other way round.
- **One transmitter.** B14 covers every port: while a client on any port owns TX, a key-up from any other port or index is refused with `busy`.
- **Shared settings are shared.** A setting that belongs to the radio rather than the slice is the same on every port, and a change is pushed to clients on all of them. `pa_enable`, `predistortion`, `band_filter`, the panadapter and the server's own limits (its maximum drive, a TX lock, a band plan) always belong to the radio. The transmit settings (`drive`, `tune_drive`, `mic_gain`, `comp`, `tx_bandwidth`) belong to the radio here too; only on a port of own transceivers (§6) may they belong to the transceiver.
- **One radio to a TCI-X client.** A slice port's manifest gives `trx_count:1` and says what `trx 0` is with `cap:slice,r,info,<value>`: `<K>,<N>` (slice K of the radio's N, also reachable as index K on the main port), `own` (this connection's own transceiver, §6) or `follow` (whichever transceiver keyed last, §6). So a TCI-X client knows it's looking at part of a radio. Logins and the operator rule (§4.3, §4.4) are the radio's, not the port's.

## 6. Own transceivers and follower ports

A server may also offer a port where each connection gets a transceiver of its own instead of a fixed slice, so several copies of a program that knows only `trx 0` (WSJT-X) can watch different bands at once. The HL2 TCI-X server does, and this section comes from it ([#4](https://github.com/g0kad/TCI-X/issues/4)). On such a port:

- **Opened and closed with the connection.** The transceiver opens when the connection does, as a copy of the main port's `trx 0`, and closes when it closes, including a client taken as lost (B20). It's the connection's alone: no other port or index reaches it except a follower port (below). The manifest says `trx_count:1` and `cap:slice,r,info,own`.
- **Receivers, not hardware slices.** A connection's transceiver needn't have a hardware receiver to itself: several on the same band may share one. So how many connections a server takes is its own choice, and its hardware limits how many bands are in use at once, not how many clients.
- **A retune never moves another transceiver.** A server places a retune as it likes (on the receiver it's on, another that covers it, its own moved if everyone else on it still fits, a free one), but never moves another transceiver off its frequency to make room.
- **Waiting for a receiver.** When there's no room for a retune, the server accepts it anyway, an exception to B3, and the transceiver waits: its audio goes on at the normal rate, silent, with no S-meter. The server pushes `rx_waiting:<trx>,true` to TCI-X clients (`cap:rx_waiting,r,bool`), so a client can tell the operator why the audio went quiet, and `rx_waiting:<trx>,false` once a receiver frees up and the transceiver takes it. While it waits, `trx` and `tune` are refused with a new `error` code, `waiting`. Refusing the retune would be worse: WSJT-X drops the connection on a refused retune and reconnects within a millisecond, so it loops.
- **Its own transmit settings.** A transceiver here may keep its own transmit settings, and the radio takes them when it keys from that transceiver: two copies of WSJT-X on different bands and antennas, each with its own drive. The manifest lists those it keeps with `cap:tx_own,r,info,<command>,…` (for example `drive,tune_drive,mic_gain,comp,tx_bandwidth`); any not listed are the radio's, as in §5. They're pushed only to that connection. The radio-wide settings and the server's limits still apply on top: a value past the maximum drive is refused with `range` and the current value echoed (B3); `cap:drive` already gives the maximum. B14 still covers every port.
- **Follower ports.** A server may offer a port for loggers that reads and tunes whichever transceiver keyed last (the main port's `trx 0` before any key), switching as another one keys. Its manifest says `cap:slice,r,info,follow` and `cap:trx0,r,info,receive_only`: it never transmits. It may retune another connection's own transceiver (a logger moving WSJT-X's band under it). While that transceiver is keyed, the follower's retune is treated as any other retune while transmitting: refused with `tx_active` only where it would leave the band or need another band filter.

## 7. Example

A FLEX-6400's two slices, with a separate SDR as a receive-only third receiver, to a TCI-X client:

```
trx_count:2;
…
cap:trx2,r,info,receive_only;
cap:trx2.vfo,rw,range,10000,30000000,1,hz;
cap:trx2.modulation,rw,enum,lsb,usb,cw,am;
cap:trx2.audio_start,w,action;
cap:trx2.audio_stop,w,action;
cap_end;
…
vfo:0,0,14074000;
vfo:1,0,7074000;
vfo:2,0,3573000;
```

A base client (WSJT-X choosing RX2) sees `trx_count:2`, uses trx 1 and never hears of trx 2.

## 8. Open points

- **New names for the main document.** `rx_waiting`, the `waiting` error code and `cap:tx_own` (§6) go into §5 when this draft does.
- **How many fixed slices:** whether a server may open and close fixed slices (§2) while clients are connected (it would send `trx_count` again), or only at start. A FlexRadio's slices are also limited by its licence and by other clients (SmartSDR) using them.
- **Split across slices:** receiving on one index and transmitting on another, as ExpertSDR's "RX2 to TX" does, is not covered.
