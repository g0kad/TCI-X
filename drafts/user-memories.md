# TCI-X draft: an operator's own memories

*Draft, not in TCI-X 1.0. First written 19 Nov 2026. kadrad-hub keeps the memories (19 Nov 2026); KADRAD, the client, doesn't use them yet. Comments welcome.*

## 1. Why

A radio's memory channels (§5.10) belong to that radio. An operator who uses several radios, at home, at a club and from a phone, has a different list in each, in each maker's format, and none at all on a radio they have never programmed.

These memories belong to the operator instead. The server keeps them for the account that logged in (§4.3), and the same list is offered on every radio that server serves. A memory holds nothing that only one radio understands, so it can be used on any radio that covers its frequency and mode.

Two things follow from that, and shape the rest:

- **The client recalls a memory**, with the ordinary commands (§6). The server stores the list and never acts on it.
- **The client carries the list between servers** (§5). An operator's servers often can't reach each other (a home network, a club behind a VPN), but the client connects to each in turn, so no server needs another's address or password.

The radio's own memories are untouched: §5.10 stays as it is, and a client may offer both.

## 2. Manifest

A server that keeps memories for the account on this connection announces:

```
cap:user_memory,rw,info,500;        it keeps up to 500 for this account
```

The number is how many memories (not counting deleted ones, §4) the account may hold; a server should allow at least 200.

The line is sent only where there's an account to keep them for, so normally only after a login. A server that needs no login may leave it out, or may keep one list for everyone who connects without one: a client can't tell the difference and needn't. The commands take no `trx` index: the list is the account's, the same on every radio the server serves.

Any role may keep memories, `listen` included: they are the account's own notes and change nothing on the radio. They need no control (§4.4) either.

## 3. Commands

| Command | Form | Notes |
|---|---|---|
| `user_memory_list` | `user_memory_list;` → one `user_memory` line per record, then `user_memory_list_end:<count>;` | Read the account's whole list, deleted records included (§4). `count` is the number of lines sent. |
| `user_memory` | `user_memory:<id>,<changed>,<state>,<name>,<group>,<rx_hz>,<mode>,<duplex_dir>,<offset_hz>,<tone_mode>,<tone_hz>,<fm_deviation>[,…];` | One record. Client → server: store it, if it's newer than the one the server holds for this `id` (§4). The server always answers with the record as it then holds it, so a client whose write lost learns what won. Server → client: also pushed to the account's other connections, on any radio the server serves, when a write is stored. |

The fields:

| Field | |
|---|---|
| `id` | The record's identity for good: 128 random bits as 32 lower-case hex digits, made by the client that created the memory. Never reused. |
| `changed` | When the record last changed, in Unix milliseconds (UTC), by the clock of the client that changed it. |
| `state` | `set`, or `deleted` (§4). |
| `name` | What the operator called it, percent-encoded, up to 64 characters. |
| `group` | A group the operator files it under (`2 m repeaters`, `HF nets`), percent-encoded, up to 32 characters, or empty. A group exists while a memory names it. |
| `rx_hz` | The receive frequency in Hz. |
| `mode` | A `modulation` name, base or §5.9. |
| `duplex_dir`, `offset_hz` | As in §5.7 and §5.10's `memory`: `simplex`, `minus` or `plus`, and the offset in Hz. |
| `tone_mode`, `tone_hz` | As in §5.7 and §5.10's `memory`. |
| `fm_deviation` | For an FM memory, `standard` or `narrow` as in §5.7's `fm_deviation`, or empty where it wasn't known when the memory was saved (the radio it was saved on couldn't say). Empty in other modes. A record that ends before this field reads as empty. |

**More fields may follow.** A server stores every field after `state` as it came and returns them unchanged, whether or not it knows them, and doesn't check them beyond the line's length (at most 1024 bytes). A client keeps and sends back fields beyond those it knows when it changes a record. So a later version, or one client, can add a field without the servers in between having to learn it.

In a `deleted` record the fields after `state` are empty: `user_memory:<id>,<changed>,deleted;`.

Errors (§5.1): `access` where there's no account to keep memories for; `range` for a malformed `id`, `changed` or `state`, a line too long, a `changed` more than a day ahead of the server's clock, or a new memory when the account already holds as many as the cap line allows. A new code, `store`: the server couldn't save the record (its disk, say); the client keeps its copy and offers it again on its next connection.

## 4. Which record wins, and deleting

Two clients may change the same memory while apart. The rule is the same everywhere, so every copy ends up alike whatever order they meet in:

- **The record with the later `changed` wins**, whole. Fields are never merged.
- With equal `changed`, a `deleted` record beats a `set` one; otherwise the holder keeps what it has.
- **Deleting is a change like any other**: the record stays, with state `deleted` and the time it was deleted. Without that, a copy elsewhere would bring the memory back. A server keeps deleted records for at least 400 days and may then drop them; a client does the same. Deleted records don't count against the cap line's number.

Since a client's clock may be wrong, a client that changes a record stamps it with the later of its own clock and one millisecond after the record's present `changed`. An edit therefore always beats what it edited. A server refuses a `changed` more than a day ahead of its own clock, so one badly set clock can't make a record unbeatable.

## 5. Carrying the list between servers

A client that keeps its own copy brings a server up to date like this, after the manifest (`cap_end`) on any connection that has the cap line:

1. Send `user_memory_list` and read to `user_memory_list_end`.
2. For each `id` the server has: if the client has none, or an older one by §4, take the server's.
3. For each `id` the client has that the server lacks or holds older: send it as `user_memory`.
4. From then on, send each change the operator makes as it's made, and take each `user_memory` the server pushes by the same rule.

Done on every connection, this makes each of the operator's servers agree with the client, and so with each other, the next time the client visits it. The client's copy belongs to the person using it, not to the account name, so it works where their account is named differently on each server.

A server holds each account's list separately and never sends one account's memories to another's connection. Whoever administers a server can read what it stores, as with anything else kept on it.

## 6. Recalling one

The client recalls a memory with the commands it would use by hand: `vfo`, `modulation`, and where the memory has them `duplex` and `tone` (§5.7), each only where the manifest offers it. They change the radio, so the usual rules apply (control §4.4, a listen-only radio, a locked dial).

A client should offer a memory only on a radio that can take it, judged from the manifest: its frequency within the radio's coverage and its mode among the radio's. One that needs a duplex or tone the radio can't set should be shown as such rather than recalled without it, since it would then transmit on the wrong frequency or without access.

**FM deviation.** Where the memory says `standard` or `narrow` and the radio offers `fm_deviation` once it's in FM, the client sets it. Where the radio doesn't offer it, the client still recalls the memory and tells the operator that the deviation is the radio's own setting, since a narrow channel worked with standard deviation is over-deviated. (Recalling with a note, rather than refusing as for a missing shift or tone, is to be judged in use.)

**`fm` and `nfm`.** A memory's mode is taken for another name only on a radio with a single FM mode, where `fm` and `nfm` can only be the same thing. On a radio that tells FM modes apart they are different modes.

A memory recalled this way leaves the radio in VFO mode. It doesn't use or change the radio's own memory mode (`memory_mode`, §5.10).

## 7. Not covered

- **Reading only what changed.** The whole list is read on each connection: a few hundred records are some tens of kilobytes. A read of what was stored since a point the server names may follow if that proves too much on a mobile link.
- **Copying from the radio's memories.** A client can read §5.10's list and save channels as memories of this kind; that needs nothing new on the wire.
- **Split-frequency pairs, filters, power and antenna.** Left out as the least portable between radios (FM's deviation is carried as what the channel needs, not as a filter). The trailing-fields rule (§3) leaves room.
- **Sharing a list between accounts** (a club's repeater list for every member).
