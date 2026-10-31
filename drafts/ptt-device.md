# TCI-X draft: a PTT device

*Draft, not part of TCI-X 1.0. First written 31 Oct 2026. The KADRAD client will take a PTT device on its local TCI port; nothing implements it yet. Comments are welcome before this goes into a minor version.*

## 1. Why

Most footswitches, and a hand mic's PTT switch, are a contact that closes while pressed. On a PC that used to mean a serial port's CTS or DSR line, which few PCs still have. A small board on the network, such as a microcontroller with a wired Ethernet port, can turn the contact into PTT for an operator's TCI-X client instead. It keys with the operator's microphone, through the client, as the client's own PTT key and space bar do.

This draft says how such a board talks to the client, and the rules that keep a lost link or a stuck pedal from leaving a radio on the air.

## 2. Where it connects

A PTT device connects to the **operator's client**, not to the radio's server. The client owns the operator's microphone audio, and the device keys with it.

The client offers it on a local TCI server of its own, such as the KADRAD client's relay port (40001 by default). A PTT device is an ordinary TCI client there: a WebSocket, text commands ending in `;`, and the base init burst ending `ready;` from the client on connecting. It sends no audio and asks for none.

**Wired, not wireless.** PTT keys a transmitter, so a device should be on wired Ethernet. A client may refuse one it can tell is on WiFi, and a client's documentation should say why. Over a wireless link, a dropped frame becomes a key-up that comes late, and a dropped connection becomes an unkey that relies on the timeout in §4.

## 3. Commands

After the client's `ready;`, the device identifies itself once:

| From | Command | Notes |
|---|---|---|
| device | `ptt_device:<name>,<version>;` | Once, after `ready;`. `<name>` is free text, percent-encoded as TCI-X text (§5.1), e.g. `ptt_device:Footswitch%20shack,1;`. Until it's sent, the client takes no `ptt` from the connection. |
| device | `ptt:<bool>;` | The contact pressed (`true`) or released (`false`), on each change, after the device's own debounce (10 to 20 ms is usual). |
| device | `ptt_alive;` | Every 200 ms while pressed. It may also be sent while released, at any interval; the client ignores it then. |
| client | `ptt:<bool>;` | Whether the client is keying for this device now. The device may show it, on an LED for example. |
| client | `ptt_refused:<reason>;` | A press the client didn't act on, with the reason as TCI-X text: `not_allowed` (§5), `wireless` (§2), `released_first` (§4), or the client's own reason. |

The device never names a transceiver or a TX audio source. The client keys its radio as its own PTT does, with its own microphone audio.

## 4. Failing safe

- **Heartbeat.** While keyed for a device, the client unkeys if no `ptt_alive;` (or `ptt:true;`) has come for 600 ms, and sends `ptt:false;` to the device.
- **Lost connection.** When the connection closes, the client unkeys.
- **Never on connect.** A press counts only after the device has said `ptt:false;` on this connection. A device that connects with the pedal already down, or reconnects with it stuck, is refused with `released_first` until the pedal comes up and goes down again.
- **One of several.** The device's PTT is one more source beside the client's own (its PTT key, the space bar). The client keys while any of them is pressed, and unkeys when the last is released.
- **The client's limits still apply.** The client's TX timeout, the server's TX lock and every refusal from the server apply as for the client's own PTT. A refusal from the server is passed to the device as `ptt_refused`.

## 5. Who may connect

A device keys a transmitter, so the client accepts `ptt` only from addresses it has been told to allow, and from none by default. Others get `ptt_refused:not_allowed;` and their connection may be closed. A client may also ask for a code set on both, as TCI-X's own logins do (§4.3).

## 6. Building one

- The contact goes between an input with a pull-up and ground: closed is pressed. On a board with long leads to the pedal, a small series resistor and a capacitor to ground keep RF out.
- Debounce in the device, and send `ptt` only on a change.
- While pressed, send `ptt_alive;` every 200 ms. If the link drops, do nothing further: the client unkeys by itself.
- Reconnect by yourself, and send `ptt_device` and then the pedal's state as it is now.

## 7. Open points

- Several pedals (a second for the voice keyer's F1, say): one device could send `ptt:<bool>,<n>;`, with `n` mapped by the client.
- The same board reading a CW paddle or a straight key, for a client's own CW keyer: a later draft.
