# TCI-X draft: RX and TX equalisers

*Draft, not part of TCI-X 1.0 and not yet implemented. First written 3 Oct 2026. It sets out what two quite different radios offer and a command shape that should cover both, so the first one built doesn't box in the next. Comments are welcome, especially from people with radios that have parametric EQs.*

## 1. What radios offer

| | Elecraft K3 / K3S | Icom IC-7100 |
|---|---|---|
| **Kind** | 8-band graphic EQ: 50, 100, 200, 400, 800, 1600, 2400, 3200 Hz | Bass and treble tone controls |
| **Range** | −16 to +16 dB per band | −5 to +5 (steps, not dB) |
| **Sets** | TX: two, one for SSB (also used in CW and DATA) and one for ESSB/AM/FM. RX: one for CW/DATA, one for voice modes. | One per mode: SSB, AM, FM, DV; RX and TX separate |
| **TX over CAT** | `TEabcdefgh;`: **set only**, all 8 bands at once. It writes the set that goes with the current transmit mode. | `1A 05` menu settings, read and set |
| **RX over CAT** | No command (menu only) | `1A 05` menu settings, read and set |

Other makes may not fit a graphic EQ. The Yaesu FTDX10 has a 3-band *parametric* mic EQ (frequency, level −20 to +10 dB, and width per band), with a second set used when the speech processor is on. The design keeps room for it (§3.3).

## 2. Principles

- **The radio is the record.** The server reads the EQ from the radio wherever CAT allows.
- **One command for every radio.** The `cap` line says what this radio has (bands, range, sets, whether it can be read), and the client draws it: 8 sliders, or two knobs.
- **No guessing.** A write-only EQ whose values aren't known is shown as unknown, not as flat.

## 3. Proposed commands

### 3.1 Manifest

```
cap:eq.<path>,<access>,eq,<min>,<max>,<step>,<unit>,<band>,<band>,…;
cap:eq.<path>.sets,r,info,<set>,<set>,…;
```

- `path`: `rx` or `tx`.
- `access`: `rw`, or `w` where the radio can't be read (K3 TX).
- `unit`: `db`, or `step` for a radio that numbers its steps without a dB value (Icom −5…+5).
- `band`: a centre frequency in Hz (`50`, …, `3200`), or a name for tone controls (`bass`, `treble`).
- `sets`: the radio's own separate EQ setups, using the radio's names: K3 TX `ssb,essb` (`essb` also covers AM and FM), IC-7100 `ssb,am,fm,dv`.

Examples:

```
cap:eq.tx,w,eq,-16,16,1,db,50,100,200,400,800,1600,2400,3200;
cap:eq.tx.sets,r,info,ssb,essb;
cap:eq.rx,rw,eq,-5,5,1,step,bass,treble;
cap:eq.rx.sets,r,info,ssb,am,fm,dv;
```

### 3.2 Values

```
eq:<trx>,<path>,<set>,<source>,<v1>,…,<vn>;
```

- Values are in band order, in the cap line's unit.
- `source`, server → client:
  - `radio`: read from the radio;
  - `sent`: the radio can't be read, so this is what this server last wrote to it;
  - `unknown`: nothing read or written yet (no values follow).
- A client write leaves `source` empty (`eq:0,tx,ssb,,0,0,2,…`) and gives every band. The server writes, reads back where it can, and pushes the result.
- **Sets the radio can only write in one mode** (the K3's `TE` writes the set for the current transmit mode): the server offers only the current set in the cap line, which comes and goes with the mode (TCI-X §4.2).

### 3.3 Later: parametric EQ

A radio with frequency, level and width per band would get its own type, e.g. `cap:eq.tx,rw,peq,<bands>,…` with three values per band. Not specified until we have such a radio's reference.

## 4. Write-only EQs

Where the radio can't be read (the K3's TX EQ), the server keeps a copy per radio and per set, so every client sees the same `sent` values. It goes wrong if someone changes the EQ on the radio's front panel, which is why the `source` field says where the values came from.
