# TCI-X draft: several slices in one radio

*Draft, not part of TCI-X 1.0. First written 5 Nov 2026, from [#4](https://github.com/g0kad/TCI-X/issues/4). Comments are welcome before this goes into a minor version.*

## 1. Why

**One radio, one transmitter, several receivers.** An SDR can run several receivers at once in the same hardware: a FLEX-6400 has two slices, a FLEX-6600 four, and a Hermes Lite 2 or ANAN can cut up to eight receivers from its hardware. Each can be tuned anywhere the radio receives, with its own mode and audio, and any one of them can be the one the transmitter uses. This draft is about those **slices**.

It isn't about several radios. A server that serves several radios gives each its own connection (its own port, its own `trx 0`), as the reference server does; nothing here changes that.

**Slices are not separate radios.** A server shouldn't present one radio's slices as several radios, each on its own connection with its own `trx 0`: they share one transmitter, one power amplifier and one set of transmit settings, which separate radios don't. Shown as separate radios, each appears to have a transmitter of its own, so TX ownership (B14) no longer covers them and two clients could key the same transmitter from two "radios"; a change to a setting they share (`drive`, `pa_enable`, `predistortion`, `band_filter`) changes all the others without their clients being told; and a client can't show the slices together. A server that wants a connection per slice for programs that know only `trx 0` uses slice ports (§5), which are the same radio.

Base TCI numbers these slices as transceivers (`trx`), as ExpertSDR does, and some base TCI clients choose one by its index. The TCI fork of fldigi lets the user pick a "receiver slice" from 1 to 8 (trx 0 to 7), and WSJT-X offers "TCI Client RX1" and "RX2" (trx 0 and 1). On ExpertSDR each index is a slice with its own frequency, mode and audio, and keying it transmits from it. A server for an SDR with several slices can offer the same, so that these clients work unchanged.

## 2. On the wire

- **Base clients too.** A server whose radio has N slices says `trx_count:N` to every client, base or TCI-X, and they take indices 0 to N−1.
- **A slice** takes the same commands as trx 0, with its own index: it tunes (`vfo`, `dds`, `if`), has its own mode, filter and the rest of trx 0's receive settings, and its own audio (`audio_start:<trx>`, RX_AUDIO blocks with `receiver` = the index).
- **One transmitter.** Keying index K (`trx:<K>,true`, `tune:<K>,true`) makes slice K the one the transmitter uses, if it isn't already (a FlexRadio's TX slice), and transmits on its frequency, with its mode and transmit settings. B14 applies across all the slices: while one client owns TX, a key-up on any index from another client is refused with `busy`. A slice that can't transmit says so (§3).
- **Shared and lasting.** A slice is the radio's state, not the connection's. Two clients choosing the same index share that slice, as two clients share trx 0 today, and it stays as it was left after its last client goes.
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
- **Shared settings are shared.** A setting that belongs to the radio rather than the slice (`drive`, `pa_enable`, `predistortion`, `band_filter`, the panadapter) is the same on every port, and a change is pushed to clients on all of them.
- **One radio to a TCI-X client.** A slice port's manifest gives `trx_count:1` and says which slice it is with `cap:slice,r,info,<K>,<N>` (slice K of the radio's N), so a TCI-X client knows it's looking at part of a radio it may also reach on the main port. Logins and the operator rule (§4.3, §4.4) are the radio's, not the port's.

## 6. Example

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

## 7. Open points

- **Slices made per connection:** a server may open a slice for each connection on a slice port, rather than offering a fixed set (the HL2 TCI-X server does). That still needs saying: how many, and what happens to the slice when its connection closes.
- **How many:** whether a server may open and close slices while clients are connected (it would send `trx_count` again), or only at start. A FlexRadio's slices are also limited by its licence and by other clients (SmartSDR) using them.
- **Split across slices:** receiving on one index and transmitting on another, as ExpertSDR's "RX2 to TX" does, is not covered.
