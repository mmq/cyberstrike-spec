# CyberStrike Server Protocol Specification

**Version 1.0 — Part I of III.** Part II is the [*Server Policy*](server-policy.md); Part III is the [*Implementer's Guide*](implementers-guide.md).

> CyberStrike was published by Simutronics and played on the GEnie online service. This specification is an independent work. It is not affiliated with, endorsed by, or derived from materials of Simutronics, GEnie or their successors. It was derived from analysis and observation of the original game client, and from the game's printed manual. All trademarks belong to their owners.

---

## 1. Introduction

### 1.1 Scope

This document specifies the protocol between the original CyberStrike client and a game server: the transport, the two phases of a session, the format of every message, and the game rules the client implements. It also states what the client leaves undefined, so that a server knows what it must provide. The client described is the DOS client, version 1.5 (1994), as used on GEnie; other versions may differ in rules.

It does not specify how a server should make the choices the client leaves open: lobby design, accounts, durability values, NPC populations and so on. Part II suggests a way to make each one. Part III is practical advice for building a server.

A server built from this document alone has been shown to support a full multi-player game with the original client.

### 1.2 The client

The client is three 16-bit DOS programs, plus map files, all shipped with the game:

| file | role |
|---|---|
| `CS.COM` | resident serial driver; stays loaded across both phases |
| `GAME.FUN` | the **Chamber** program: logon, lobby, launch. Version 1.05 |
| `POD.FUN` | the **game engine**. Version 1.50 |
| `C1.CBD`–`C3.CBD` | city map files |

The client reaches the server through a serial port. Only one version of the client is known to exist.

### 1.3 Conformance language

The words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as follows:

- **MUST / MUST NOT**: the client will fail, crash, desynchronize or misbehave visibly if the server does otherwise.
- **SHOULD / SHOULD NOT**: the game works otherwise, but a feature is lost or degraded.
- **MAY**: an option that works with the client.

These words apply to servers. The client's behavior is described, not required.

### 1.4 Confidence grades

Every rule in this specification carries a grade saying how firmly it is known.

| grade | meaning | relying on it |
|---|---|---|
| **confirmed** | The client has been seen behaving this way in operation. | Safe. |
| **established** | Follows from the client's logic with every case accounted for, but has not yet been seen in operation. | Safe, but expect occasional surprises. |
| **inferred** | The structure is known (field count, sizes, encodings); the meaning is a best interpretation. | Implement defensively. |
| **server-defined** | The client has no rule here. The server must choose a value or behavior; Part II suggests one. | Your decision. |
| **unknown** | The message or field exists but is not understood. | Do not send it. |

A grade applies to the paragraph, table row or list item it ends. Where a whole section shares one grade, it is given once, in italics, under the heading.

Where the game's manual describes a rule that the client does not implement, this document says so ("the manual's rule") and grades it **server-defined**: a server implements it, or not, by choice.

### 1.5 Glossary

| term | meaning |
|---|---|
| **pod** | a player's walking combat vehicle |
| **net id** | the number identifying an entity on the wire; assigned by the server (§6.2) |
| **Chamber** | the pre-game phase: logon, lobby and launch, run by `GAME.FUN` |
| **city** | a joinable game, played on one map. A **map** is the `C<n>.CBD` file supplying its geometry |
| **block** | one 600 ft × 600 ft square of the city; the city is 8 × 8 blocks. Also called a cell when describing positions |
| **home corner** | the corner block where a team's pods spawn (§6.3) |
| **power tower** | a structure that supplies power to nearby pods and claims its block for its team. A **MegaTower** is a larger variant |
| **territory** | the blocks a team controls; they earn score (§9.2.6) |
| **module** | an add-on installed in a pod's **bay**; each bay has a level that limits which modules fit (§9.3) |
| **guppy** | a flying delivery vehicle that brings ordered towers and modules (§9.4) |
| **Sentinel** | a stationary NPC gun emplacement (§9.5) |
| **Muncher** | a flying NPC that eats towers (§9.5) |
| **bounty** | a value on every pod that grows while it lives and is paid to those who destroy it (§9.2) |
| **multiplier** | a per-player factor applied to bounty and territory score (§9.2.5) |
| **bay points**, **battle marker** | the two counters that drive bay upgrades (§9.2.7) |
| **tick** | one step of the client's scoring clock: at least 1.0436 s, typically about 1.05 s (§9.2.4) |
| **drop state** | the state of a pod waiting to be released into the city (§7.4) |
| **spawn grace** | a period after spawning during which a pod takes no damage (§9.1.5) |
| **tag** | a server-chosen byte in some server messages, echoed back by the client (§4.2) |

### 1.6 Notation and message names

#### Byte values

Bytes are written in hexadecimal without a prefix (`43 53`) inside frame dumps and with a `0x` prefix (`0x10`) in text. A character in backquotes (`` `D` ``) stands for its ASCII byte (`0x44`).

#### Messages, types and subtypes

In the game phase every message begins with a **type** byte, an ASCII letter. Some types begin a family of messages, and for those the next byte is a **subtype**, also a letter. For example, every message beginning `` `D` `` reports or sets progression, and the subtype says which: `` `S` `` for score, `` `B` `` for bays.

Every message has a **name**, used throughout this document, and a **code**, used where the bytes matter:

| name | code | meaning |
|---|---|---|
| Score Status | `D S` | type `D`, subtype `S` |
| Kill Notice | `E K` | type `E`, subtype `K` |
| Pod State | `b` | type `b`, no subtype |

Prose uses the name: "the server sends a Score Status". The code appears in the message catalog (§8), in frame dumps, and in parentheses on first use in a section: "a Score Status (`D S`)". The full list of names is the index in §8.2.

#### Direction

**S→C** is server to client, **C→S** client to server. The same type letter can mean different things in the two directions; such messages have a separate name for each:

| code | S→C | C→S |
|---|---|---|
| `I` | Damage | Hit Report |
| `J` | Destroy Entity | Pod Died |
| `M` | Spawn Tower | Tower Order |

#### Field types

Message layouts use these field types, defined in §4.3:

| type | size | meaning |
|---|---|---|
| `b10` | 1 byte | small unsigned value, offset by `0x10` |
| `b88` | 1 byte | signed value, offset by `0x88` |
| `num` | 1–8 bytes | variable-length signed integer |
| `str` | 1 + n bytes | length (`b10`) followed by n bytes of text |
| `cell` | 1 byte | a map cell (§6.1) |
| `char` | 1 byte | a literal ASCII letter |

Positions are written as `cell, x:num, y:num, z:num` (§6.1). Angles are in tenths of a degree, 0–3599.

#### Chamber notation

In the Chamber phase (§5) the protocol is text. `\r` is a carriage return (`0x0d`), `\n` a line feed (`0x0a`). A **command line** is written `05 "C\r"`: the byte `0x05` followed by the text.

---

## 2. Overview

### 2.1 The game

CyberStrike is a team game for up to sixteen players in a city of 8 × 8 blocks. Each player controls a **pod**, a walking vehicle armed with a gun. Pods belong to one of four teams (Red, Blue, Green, Black) of up to four players, each starting in its own corner of the city.

A pod runs on power, which drains continuously and is replenished by being near a friendly **power tower**. Players order towers to be delivered by their team's **guppy**; a tower also claims its block, and each block a team controls alone adds to its players' score. Every pod carries a **bounty** that grows while it survives and is paid to the players who destroy it. Score is multiplied by a **multiplier** that rises the longer a player survives. Players earn better **bays** over time and fill them with **modules**: speed, shields, mines, artillery and more, also delivered by guppy.

The city also holds NPCs: **Sentinels**, gun emplacements that fire at nearby players, and **Munchers**, which eat towers.

### 2.2 Architecture

The client speaks over a serial port. A server reaches it by bridging that port to a TCP connection (in practice, through an emulator's serial-to-TCP feature).

A session runs two different client programs, one after the other, over **one** continuous connection:

```
   [ Chamber phase ]   GAME.FUN   ASCII text, line-oriented
            |
            |  the Chamber program exits; the game engine starts
            v
   [ Game phase ]      POD.FUN    binary frames
            |
            |  the player quits; the Chamber program starts again
            v
   [ Chamber phase ]   ...
```

The connection is never re-established between phases. The server sees one byte stream that changes language partway through. *Confirmed.*

The Chamber protocol is the generic logon protocol of the GEnie service, followed by a small command language (§5). The game protocol is a set of binary messages framed with a checksum (§4).

### 2.3 A session

1. The client connects and logs on (§5.1).
2. The server presents a lobby, rendered as text; the player chooses a game (§5.2–5.3).
3. The server tells the client which map to load and launches the game engine (§5.4).
4. The server starts the game session; the client's pod drops into the city (§7.1).
5. Players move, fight, order towers and modules, and score (§9). Pods that are destroyed respawn (§7.5).
6. The player quits; the client returns to the Chamber on the same connection (§5.5), and may play again.

### 2.4 Division of labor

The client is one half of a client–server pair whose server half no longer exists. What the client does for itself, the server need not; what it leaves out, the server must supply.

**The client does:**

- simulate its own pod: movement, physics, power drain, collisions;
- detect hits by projectiles it originates, and apply damage to its own pod from projectiles it simulates (§9.1);
- decide which client simulates each NPC's fire (§9.5);
- compute territory control from the towers it has been sent (§9.2.6);
- run its own player's scoring and progression while a session lasts (§9.2);
- draw everything.

**The server must:**

- run the logon and the entire lobby (§5);
- assign every entity's net id (§6.2);
- relay each pod's state, shots and chat to the other clients;
- apply damage between players (§9.1.4);
- set and reset every pod's bounty, judge earned kills, and keep score across sessions (§9.2);
- decide the durability of towers, Sentinels and Munchers (§9.1.4);
- create NPCs and carry out deliveries and module effects (§9.3–9.5);
- choose every value listed in §10.

---

## 3. Transport

*Confirmed unless marked.*

The client uses a resident serial driver, a BIOS INT 14h handler. A server connects to it through a byte stream bridged to that serial port.

- **The link MUST be 8-bit clean (8 data bits, no parity).** Game-phase checksum bytes always have bit 7 set. A 7-bit link passes the Chamber phase, which is ASCII, but corrupts every game frame.
- The client's receive buffer holds 4096 bytes and survives the change from the Chamber program to the game engine, so bytes sent while the engine is loading are not lost. **When the buffer is full, every further byte is lost** until the client reads from it. *Established.* This limits what a server may send before the game engine starts reading (§7.1).
- The game engine processes at most 5 frames per receive call, at about 10 calls per second. A server that sends more than about 50 frames a second builds a backlog. *Inferred.*

### 3.1 The no-traffic watchdog

*Established.*

Once the game engine has received its first frame, every frame it receives restarts a 30-second watchdog, timed in whole seconds. If no frame arrives, the client escalates every 5 s:

| seconds without a frame (about) | the client |
|---|---|
| 30 | displays `** Network problems...retrying!` |
| 35 | displays it again |
| 40 | beeps, sends the modem hang-up text `+++` and `ATH0\r` as raw bytes, and displays `** Network problems...DISCONNECTING!` |
| 45 | repeats the hang-up text and message |
| 50 | the game engine exits and the Chamber program starts again, as in §5.5 step 5. No `QUIT\r` is sent |

A frame arriving at any point before the game ends displays `Network recovered!` and resets the watchdog.

**A server MUST send each client in a game at least one frame every 30 s.** The client sends a Ping about every 4 s (§8.7), so a server that answers every Ping, as it must, meets this without further effort. The hang-up text arrives outside any frame and is skipped by a receiver that scans for `43 53`.

---

## 4. Framing and data types

### 4.1 Frame format

*Confirmed.*

In the game phase every message, in both directions, is carried in one **frame**:

```
+-----+-----+-----+-----+-----+------------------------+-------+
| 'C' | 'S' | CK  | ~CK | LEN | payload (1–234 bytes)  | [END] |
+-----+-----+-----+-----+-----+------------------------+-------+
  43    53
```

| field | size | value |
|---|---|---|
| `'C' 'S'` | 2 | the bytes `43 53`; marks the start of a frame |
| `CK` | 1 | checksum: `((LEN + sum of payload bytes) & 0x7f) + 0x10` |
| `~CK` | 1 | the bitwise complement of `CK` |
| `LEN` | 1 | `5 + payload length + 0x10`: the frame length from `'C'` to the end of the payload, offset by `0x10` |
| payload | 1–234 | the message: type byte, then (for some types) a tag (§4.2), then the fields |
| `END` | 0 or 1 | terminator: `0d` or `0e` from the server, **required**; `0a` from the client, optional |

**Delimiting.** The two directions are delimited differently. *Confirmed.*

- **Server to client: every frame MUST end with `0d` or `0e`.** The client reads a frame up to its terminator, so a frame without one runs on into the next. Once 512 bytes have accumulated without a terminator, the client stops with the fatal error `PacketCnt > 512` (*established*).
- **Client to server:** a server MUST find the end of a frame from `LEN`, not from a terminator. The client omits the terminator on some frames, so frames may follow each other with no byte between them. After `LEN − 0x10` bytes, the server skips one terminator byte if one is present.

**Invalid frames.** The client silently discards a frame whose start bytes, `~CK` or checksum do not match. A server SHOULD do the same with the client's frames and resynchronize on the next `43 53`.

**Frames that pass the checks but do not decode.** About one real client Pod State in a thousand has a valid checksum but does not decode as its layout; the cause is unknown. A server SHOULD drop such a frame and carry on, never disconnect for it. It SHOULD also ignore bytes after the last field of a client message. *Confirmed.*

**Which terminator.** `0d` and `0e` are equivalent to the client. Server frames conventionally use `0d`. *Established.*

**Size limit.** `LEN` is one byte, so a payload is at most 234 bytes. Variable-length fields must be sized against this limit.

**Forbidden bytes.** Payload bytes MUST NOT be `0a`, `0d` or `0e`: the client would take them as the end of the frame. The encodings in §4.3 keep almost every field at `0x10` or above for this reason; the exceptions are `b88` (§4.3) and Kill Notice's `verdict` byte (§8.7).

**Resynchronization.** The client occasionally sends a lone `05` between frames as a keep-alive. A receiver that scans for `43 53` skips it.

**Message type.** The first payload byte is the message type, an ASCII letter from `A` to `c`.

**Example.** A Release Pod (`A`) message, server to client, releasing pod 0, with tag `20`:

```
43 53 19 e6 18 41 20 10 0d
```

| bytes | field | value |
|---|---|---|
| `43 53` | start | |
| `19` | CK | `((0x18 + 0x41 + 0x20 + 0x10) & 0x7f) + 0x10 = 0x19` |
| `e6` | ~CK | complement of `19` |
| `18` | LEN | `5 + 3 + 0x10` |
| `41` | type | `A` |
| `20` | tag | (§4.2) |
| `10` | pod id | `b10`: 0 |
| `0d` | END | |

### 4.2 The tag

*Confirmed.*

A server-to-client message whose type is an **uppercase** letter has one extra byte, the **tag**, immediately after the type byte and before any subtype. Lowercase types have no tag. Client-to-server messages never have a tag, whatever their case.

The tag's value is chosen by the server and has no meaning to the client, which returns it in an Acknowledge (`d`) message. Any value from `0x10` to `0xff` is valid; a constant works. A server that wants to match Acknowledges to the messages they answer (to measure round-trip time, for instance) uses a different tag for each message, such as a counter. Message layouts in §8 omit the tag.

### 4.3 Field encodings

Three encodings are used for numbers, and each field uses exactly one of them. The message catalog gives the encoding of every field. Using the wrong encoding for a field is the most common implementation error.

#### `b10` — small unsigned value

*Confirmed.*

```
wire = value + 0x10
```

Range 0–239. Used for most single-byte values, **including string lengths**. An unencoded length makes the client read far past the end of the message.

#### `b88` — signed value

*Confirmed.*

```
wire = value + 0x88
```

Range −136 to 119. Used for forward speed and a few other signed fields. Values below −120 encode below `0x10`, and three of them are forbidden bytes (§4.1): **a server MUST NOT send −126, −123 or −122** in a `b88` field.

#### `num` — variable-length integer

*Confirmed.*

Used for entity ids, positions, angles and other values that may be large or negative. The first byte carries the sign, the value modulo 15, and how many further bytes follow. Each further byte is a base-240 digit, least significant first.

```
encode(v):
    sign   = 1 if v < 0 else 0
    v      = |v|
    low    = v mod 15
    digits = base-240 digits of (v div 15), least significant first   (0 to 7 digits)
    emit ((low << 4) | (len(digits) << 1) | sign) + 0x10
    emit each digit + 0x10

decode:
    h      = first byte − 0x10
    sign   = h & 1
    count  = (h >> 1) & 7
    v      = h >> 4
    scale  = 15
    repeat count times:  v += (next byte − 0x10) × scale ;  scale ×= 240
    if sign: v = −v
```

| value | bytes |
|---|---|
| 0 | `10` |
| 5 | `60` |
| 145 | `b2 19` |
| 240 | `12 20` |
| 3000 | `12 d8` |
| −20 | `63 11` |

Every byte of a `num` is at least `0x10`. The length of a `num` depends on its value: a field that changes from one byte to two shifts every field after it, so a decoder MUST read each `num` completely before reading the next field.

#### `str` — text

*Confirmed.*

A `b10` length, then that many bytes. There is no terminating zero.

---

## 5. Chamber phase

The Chamber phase is text. The client behaves as a terminal with a line editor and some buttons; the server drives it with text and a few command lines.

### 5.1 Logon

*Confirmed unless marked.*

The logon is the generic GEnie service protocol, not specific to CyberStrike. The client follows a fixed script:

```
client: HHH                    no terminator
server: U#=                    the client waits for "U#"
client: \r                     a bare CR
client: <account>\r            the account line
client: <Not Telling!>\r       always, shortly after the account line
server: \r\n*                  the client waits for "*", the service prompt
client: M 1380;3\r             later, when the player presses GO
```

- The client's modem dialing is handled locally, so **the first bytes the server receives are `HHH`**, with no terminator. A server scans the stream for `HHH` and ignores anything before it.
- The server needs to send only `U#=` and, after the account line, `*`. The client does not wait for anything else; other text (such as `CONNECT` or `PASSWORD:`) is displayed and otherwise ignored.
- `*` may be sent before or after `<Not Telling!>` arrives. *Before: confirmed. After: inferred.* The `<Not Telling!>` line carries nothing and is ignored.
- **The first non-empty line is the account line.** The bare CR before it produces an empty line.
- **The account line** is `NAME,PASSWORD` for any configured name except `JOINGENIE`, and `NAME` alone for `JOINGENIE` (compared without regard to case). *`JOINGENIE` form confirmed; password form inferred.*
- **The name is set by the player** in the client's configuration. It is the player's GEnie login, not a callsign. An unmodified installation has `JOINGENIE`, which is probably the name GEnie used for signing up new members rather than one players entered the game with. *Inferred.* Whether a server accepts `JOINGENIE`, and whether it lets more than one connection use the same name, is server-defined (§10).
- **The callsign is separate from the account name.** On the original service a player chose a callsign after logging on, and it was tied to their GEnie account; the case the player chose was kept. The client has no step for choosing one: it is told its own callsign in Start Session (§8.4) and other players' in Introduce Pod (§8.9). How a server obtains and assigns callsigns is server-defined (§10). *Server-defined.*
- `M 1380;3` is the GEnie page command for CyberStrike. It means the player has asked to enter the game.

### 5.2 The terminal and command mode

The Chamber client draws what the server sends as text, reading 7 bits of each byte. *Inferred unless marked.*

- Printable characters are drawn. **CR** starts a new line, **BS** moves back, **BEL** beeps.
- **LF, TAB, ESC and every other control byte are discarded.** There is no ANSI support. Lines end with CR.

A line is treated as a **command** only after the server sends an arming byte. Without one, even a well-formed command is just printed. *Confirmed.*

| byte | effect |
|---|---|
| `0x01` | arms command mode and starts the command with `+` |
| `0x05` | arms command mode for the next CR-terminated line |

**An armed line MUST be shorter than 512 bytes.** The client copies it without a bounds check.

The commands a server uses (`05` + text + CR):

| command | effect | grade |
|---|---|---|
| `A` | show the Chamber buttons (PLAY, more, QUIT) and a "CYBERSTRIKE HUB" banner | confirmed |
| `B` | show the GEnie service buttons (GO, SETTINGS, BYE) | confirmed |
| `D<digit>` | set the text color for what follows, from a seven-entry palette. Digits 7–9 are black on black | inferred |
| `E <map> <rev>` | prepare to launch on a map (§5.4) | confirmed |
| `C` | launch the game engine (§5.4) | confirmed |
| `PRESS RETURN` | version check: the client replies `CSTTBT 105 150\r`, or a bare `\r` if its game engine is missing or invalid. Hides the buttons; follow it with `A` | inferred |

The client's command table has five more entries: `W`, `Z` and `+` draw operator load meters in display modes no server command can reach, and `X` and `Y` are debugging lines. A server never needs them and SHOULD NOT send them.

### 5.3 Player input

*Inferred unless marked.*

The lobby itself, everything the player sees and does before launching, is provided by the server. The client contributes a line editor, buttons and the display.

**Typed text** arrives as a whole line ending in `\r`, never a keystroke at a time. ESC and Ctrl-X are exceptions, sent at once on their own. The client displays sent lines itself by default (the player toggles this with Alt-E), so a server SHOULD NOT echo them back.

**Buttons** put text into the input line:

- `/PLAY`, `/QUIT`, `/STATUS`, `/SCORE`, `/WHO`, `/CITY` are sent at once. *`/PLAY` and `/QUIT` confirmed.*
- `/WHO `, `/CITY ` and `/PLAY ` with a parameter are typed into the line for the player to finish.
- The two buttons that take a city letter first send a bare `/CITY`, then type `/CITY ` or `/PLAY ` into the input line. So the server MUST answer a bare `/CITY` with a list of cities identified by letter.
- **The reply to that bare `/CITY` MUST be plain text, with no command lines.** If it contains one (a color change, for instance), the second half of the button's action is lost. Color changes at any other time are harmless. *Confirmed; the reason is unknown.*
- Buttons add to the input line without clearing it. If the player has typed part of a line, the button sends that text followed by its own, for example `draft/CITY`. *Confirmed.*

**The command language is the server's.** The manual documents `/HELP` and `/RESUME`, but neither appears anywhere in the client. They are typed by the player as ordinary lines. A server therefore interprets typed lines as commands in its own right; the buttons are shortcuts into that language, not its whole vocabulary. *Established.*

### 5.4 Launch

*Confirmed unless marked.*

```
server: 05 "E <map> <rev>\r"
client: /GO <status>\r
server: 05 "C\r"                 only after /GO 0
        (the Chamber program exits and the game engine starts, loading the map)
```

| `/GO` status | meaning |
|---|---|
| 0 | ready |
| 1 | no file for that map |
| 2 | the map file's header does not match `<rev>` |
| 3 | not enough memory |

- **`<map>` names a file and nothing else.** It selects the client's `C<map>.CBD`. Any number works if that file exists on the client, including generated maps; a missing file is reported as `/GO 1` before launch. A map file must be valid (correct checksum, geometry in every block) to load. *Inferred.*
- **`<rev>` MUST be a single digit, 0–9.** The client writes `<rev>` as two digits after the text `CSCBD2` and compares the result with the map file's 8-byte header, but only its first 7 bytes, so only the tens digit of `<rev>` is checked. The shipped maps' headers have 0 there, so any `<rev>` from 0 to 9 passes, and 10 or more gives `/GO 2`. *Inferred.*
- The map number reaches the game engine through the resident driver, not the connection, so the server does not repeat it after launch.

### 5.5 Returning after a game

When the player quits, the game engine exits and the Chamber program starts again **on the same connection**, without a new logon.

The sequence:

1. The client sends a Quit Request (`C`, no fields). Before it, the client sends its final status reports (§9.2.10); they need no reply. *Established.*
2. The server answers with Quit (`C`). This is an uppercase message and so carries a tag, like any other. *Established.* A server MUST build it as a new message rather than echo the client's frame: the client's frame has no tag, and on a one-byte payload the tag's position is where the terminator would be, so an echoed frame never ends. *Established.*
3. The server stops sending game messages. Anything sent after this point is drawn as text by the Chamber program.
4. The client sends `QUIT\r` twice, as raw text. The first may arrive in the same read as the last game frames, so the server scans for it. *Confirmed.*
5. The Chamber program starts with no buttons and sends nothing. *Inferred.* Once it has had time to start, the server sends `05 "A\r"` to restore the Chamber buttons. Whether the Chamber program discards bytes that arrive while it is starting is **unknown**; a pause of a few seconds is enough. *Confirmed.*

The player may then launch another game without logging on again. *Confirmed.*

The server may also end a game unprompted by sending Quit; the client returns to the Chamber in the same way. *Confirmed.*

---

## 6. World model

### 6.1 Coordinates and orientation

*Confirmed unless marked.*

A city is an **8 × 8 grid of 600 ft blocks**, 4800 ft square. Positions are in whole feet, and are carried on the wire as a block (the **cell**) and offsets within it:

```
cell = ((cellX << 4) | cellZ) + 0x10        one raw byte; X and Z in the two halves
x    = num(worldX − cellX × 600)
y    = num(worldY)                          absolute altitude, not relative to the cell
z    = num(worldZ − cellZ × 600)
```

- **Altitude is absolute** and has no cell component.
- **Pods can be outside the city.** Below 0, the client sends cell 0 with a **negative** offset, never cell −1. A server that re-encodes a position MUST keep the offset signed rather than clamping the coordinate. Positions beyond 4800 ft have never been seen.
- **Altitude does not mean flight.** Pods stand on rooftops at about 212 ft. Whether a pod is airborne comes from its state (§7.4), not its height.

**Orientation** (`orient`, and a shot's `yaw`) is in tenths of a degree, 0–3599. A pod facing θ moves along (sin θ, cos θ) in (x, z): 0 faces +Z, 900 faces +X. **Pitch** uses the same unit: level is 0, and upward angles wrap below 3600 (an upward lob is about 3332).

### 6.2 Entity ids

*Confirmed unless marked.*

Every entity (pod, tower, mine, NPC, vehicle, dropped module) has a **net id**, assigned by the server.

**The id field depends on the message.**

- Pod messages carry the id as one `b10` byte: Release Pod, Introduce Pod, Remove Entity, Pod State, Pod Flags and Set Bounty. **A pod's id is therefore 0–239.** *Established.*
- Every other message carries the id as a `num`, and the client stores it as a 16-bit value (*established*). Ids of 4096 and above have been used for Sentinels, and above 8191 for towers and dropped modules.

So only pods need ids below 240. Towers, modules, mines and NPCs SHOULD use larger ids, leaving the byte range for pods.

**Reuse.** The client keeps every entity in a list keyed by id, with no duplicate check, and the first match wins. It never removes entries, and the messages that create towers and dropped modules ignore an id that already exists, while Introduce Pod creates a duplicate.

- **A net id MUST NOT be reused while any connected client might still hold it.** A reused id is invisible or doubled on a client that saw the old one. *Confirmed.*
- A client whose game engine started after the id was released has a fresh list, so reusing the id is safe for it. *Inferred.*

Vehicle ids also select the vehicle's kind (§8.13): below 260 a guppy (only 250–259 fly above ground), 260 and above a Muncher.

### 6.3 Players, teams and home corners

*Confirmed.*

There are four teams, numbered 0–3 and displayed as Red, Blue, Green and Black. A team has at most four players.

Each player has a **slot**: an index within the team (0–3) and the team, carried in one byte as `(index << 3) | team`. **Each player MUST have a unique (team, index) pair.** The client's roster panel places each name by its slot, and two players with the same slot overwrite each other.

Each team spawns in its **home corner** block, given in (cell X, cell Z):

| team | 0 Red | 1 Blue | 2 Green | 3 Black |
|---|---|---|---|---|
| home corner | (0, 0) | (7, 0) | (0, 7) | (7, 7) |

### 6.4 City settings

Start Session (`B`, §8.4) gives the client three settings of the city it is playing:

| setting | effect |
|---|---|
| maximum multiplier | the highest score multiplier a player can reach (§9.2.5). MUST be at most 51 (§9.2.6) |
| tower output | 0 low, 1 normal, 2 high: how much power towers supply |
| maximum module level | 0–5: the highest module level the purchase menu offers |

Everything else that distinguishes one city from another (block values, weather, NPCs, which modules are stocked) is controlled by the server through ordinary messages.

---

## 7. Session and pod lifecycle

### 7.1 Starting a session

*Confirmed unless marked.*

```
  server                                   client
    |---- Start Session (B) ----------------->|   at once after launch
    |<--- Pod Flags (S) ----------------------|   in bursts of 4: "my pod exists"
    |<--- Pod State (b), state 4 -------------|   the pod waits in drop state
    |---- Release Pod (A) <own id> ---------->|   repeat while it still reports state 4
    |<=== Pod State (b) =====================>|   ~2/s moving, ~1/s still, both ways
    |<--- Acknowledge (d) --------------------|   returns tags, in batches
```

1. **Send Start Session immediately after launch.** The game engine processes messages from its very first frame. A bare CR the client sends some seconds later means "no session yet"; it is not a readiness signal, and waiting for it wastes about ten seconds. Only the first Start Session takes effect and later ones are ignored, so the server MAY repeat it until the client's first frame arrives. **But the engine reads nothing while it loads**, so everything sent between launch and the client's first frame accumulates in the 4096-byte receive buffer, and bytes beyond it are lost (§3). The server MUST keep the total it sends in that interval below 4096 bytes. *Established.*
2. **The first drop has no request.** At session start the client sends no Drop Request; its pod simply reports state 4. The server answers with Release Pod naming the pod, and keeps answering while the reports still say 4. Release Pod does nothing to a pod not in drop state.
3. Send the player's status: Bay Status, then Score Status (§9.2.11). Until it receives a Bay Status the client runs none of its per-tick scoring (§9.2.3).
4. Introduce the other players (§7.2) and give the client the world as it stands (§7.3).
5. Answer the client's Ping at once (§8.7).

### 7.2 Introducing players

*Confirmed unless marked.*

When two players are in the same city, each client must be told about the other's pod.

1. Wait for the newcomer's first Pod State: Introduce Pod carries a full position, and needs one.
2. Send each existing player's Introduce Pod (`G`) to the newcomer, and the newcomer's to each existing player, **once per pair**. Introduce Pod creates a pod with no duplicate check, so a second one makes a second pod.
3. **Follow every Introduce Pod with Release Pod naming the same id, to the same client.** Introduce Pod leaves the new pod in drop state, and a pod in drop state ignores every Pod State: without the release, the other player stays frozen where they were introduced. *Established.*
   **Exception: a pod that is dead** (between its Destroy Entity and its owner's Drop Request) is introduced at its last reported position and is **not** released. Every other client's copy is waiting in drop state, and the Release Pod that answers the Drop Request goes to every client holding the pod, the newcomer included (§7.5). Releasing it early would drop the newcomer's copy while everyone else's waits. *Inferred.*
4. From then on, relay each pod's Pod State to the other clients (§8.24).

### 7.3 Late joiners

A client knows only what it has been sent. A client joining a game already in progress must be given the world as it stands.

- **Every live entity's creating message, exactly once per client**: Spawn Tower, Spawn Sentinel, Spawn Mine, Spawn Module, and the create of each guppy. The joining client replays the creation as it first happened: a tower or Sentinel falls from 300 ft again. *Confirmed for towers and Sentinels.*
- **The territory grid** (Block Classes `T`), once. *Confirmed.*
- **The weather**, as its current state (Ambient `R` subtypes 0 and 2), once, and every change after. *Confirmed for the palette; inferred for rain.*

**Guppies.** Every client counts a guppy's legs by its own Leg Arrived reports (§9.4). A client is out of step when it has missed the earlier legs (a joining client), or when it has not yet reported the leg that another client's report has just acted on (§9.4.1). Such a client must be brought into step: *confirmed for a joining client, inferred for the second case*

1. Send the joiner the guppy's create Move Vehicle, at its home corner. If the guppy is still on its create leg, the joiner counts like everyone else, and nothing more is needed.
2. Otherwise, send the joiner no other Move Vehicle for that guppy until it reports. A destination sent before the create leg is reported replaces it, and is then never reported.
3. On each report from the joiner: if the guppy has moved on since the leg last given to this client, send its **current** destination and wait for the next report. Otherwise the joiner is in step. If the guppy is at home, its next report is the next leg; if in flight, this report is the current leg's arrival.
4. While a client is being brought into step, its reports drive nothing; the other clients' reports do.

A client that is out of step but already holds the guppy's create starts at step 2: the leg last given to it is the leg it is still flying. No timers are involved. Without this procedure, a client's reports are out of step, and once the original clients leave, the guppy is stranded.

**Munchers.** A Move Vehicle for a Muncher that a client does not know is a create: it makes a new, stray Muncher that heads for the city origin. So Muncher messages MUST go only to clients that hold its create. Give a joining client its own create and an immediate dispatch, starting where the current leg began, or at the tower itself if the Muncher is eating. From then on, treat that client as a guppy joiner from step 3 above: until it is in step its reports drive nothing; on each report, send it the Muncher's current destination if the Muncher has moved on since the leg last given to it, otherwise it is in step. If the Muncher is eating when the joiner comes into step, send it the Muncher Attack. *Confirmed for a joiner arriving while a Muncher is eating.*

### 7.4 Drop and flight states

*Confirmed.*

Every Pod State carries a **state** byte, a small state machine:

```
4 waiting to drop  →  1 falling  →  2 touching down  →  0 on the ground
```

- A pod in state 4 is in **drop state**. It ignores every Pod State sent about it, and is released by Release Pod.
- Whether a pod is dropping or airborne is read from its state, never inferred from its altitude.
- When relaying, the server sends only states 0 and 1 (§8.24).

### 7.5 Death and respawn

*Confirmed unless marked.*

```
  dying client                server                   other clients
      |--- Pod State (power < 1) -->|
      |--- Pod Died (J) ----------->|
      |<-- Destroy Entity (J) id ---|--- Destroy Entity (J) id -->|
      |    (~9 s fly-over; reports state 4 from ~1.2 s)
      |--- Drop Request (A) ------->|
      |<-- Release Pod (A) id ------|--- Release Pod (A) id ----->|
```

1. When its pod's power falls below 1, the client sends a final Pod State and then Pod Died (`J`, no fields).
2. **The server sends Destroy Entity naming the pod to every client, including the dying client.** The dying client shows its death camera but does not complete the death until it receives this. The scoring side of a death (payouts and bounty reset) is in §9.2.9.
3. The dying client plays a fly-over of about 9 seconds. From about 1.2 s after the death its pod reports state 4.
4. **The server MUST NOT release the pod in answer to those state-4 reports.** Doing so drops it in the middle of the fly-over.
5. At the end of the fly-over the client sends a Drop Request (`A`, no fields). The server answers with Release Pod naming the pod, **to every client holding it**: the pod's copies on other screens also wait in drop state. The pod then drops at its home corner.

On its own death the client also resets its own state silently (§9.2.9).

### 7.6 Leaving

When a player leaves the city, the server sends Remove Entity (`H`) naming the pod to every other client. *Confirmed.* It MUST NOT use Destroy Entity for this: that makes the pod explode and respawn.

### 7.7 How a session ends

*Inferred unless marked.*

| ending | what the client sends | what the server knows |
|---|---|---|
| **quit** | Score Status, Bay Status, Score Status, Quit Request (the reports only once scoring is active); after the server's Quit, `QUIT\r` twice (§5.5) | everything, up to the moment |
| **line drop** (crash, closed window, lost connection) | nothing | the last Score Status (up to about a minute old), the last Pod State (about a second old), and its own records |
| **death** (not an ending) | Pod Died, fly-over, Drop Request (§7.5) | the next Score Status |

The quit sequence's reports are built from live values and can be saved like any other.

**Restoring a player after a line drop.** No message resumes a session; a server that wants to restore a player replays ordinary messages after a new launch:

| restorable | how |
|---|---|
| bays and battle marker | Bay Status, first |
| SCORE, multiplier, bay points | Score Status |
| installed modules | Module Operation install (§8.19), one per module, highest level first, after the pod has landed |
| bounty | Set Bounty |
| position | an own-pod Pod State (§8.24) |

Not restorable: power and armor (a restored pod starts at 3000), module charges and on/off settings, which bay each module occupies (the client places modules itself), and the multiplier's step countdown.

### 7.8 State diagrams

#### One connection

```
 connect
   |  logon (§5.1): HHH → U#= → account line → <Not Telling!> → * → M 1380;3
   v
 CHAMBER  (text; GAME.FUN)
   |  arm command lines with 0x01 or 0x05 (§5.2)
   |  lobby (server-defined) — the player chooses a game
   |  launch: 05 "E <map> <rev>" → /GO <status> → 05 "C" if status 0 (§5.4)
   v
 GAME  (binary frames; POD.FUN; same connection)
   |  the pod's life, below
   |  Quit Request → Quit → stop game traffic → QUIT\r ×2 (§5.5)
   |  connection lost → the session ends here (§7.7)
   v
 CHAMBER again: after a few seconds, 05 "A" restores the buttons
```

#### One pod

```
 JOIN    Start Session at once, repeated until the client answers (§7.1)
   |     Introduce Pod + Release Pod for each other pod, and this pod's to them (§7.2)
   |     the world: towers, mines, modules, Sentinels, vehicles, territory grid, weather (§7.3)
   |     Bay Status, then Score Status (§9.2.11)
   v
 DROPPING      state 4; positions ignored. Release Pod while it reports 4
   v
 FALLING (1) → TOUCHING DOWN (2) → ON GROUND (0)
   |     spawn grace until the pod leaves its home box or 2 minutes pass (§9.1.5)
   |     in play: relay Pod State and Shot; relay hits as Damage; push bounties;
   |     answer Ping; carry out orders and module effects (§9.1–9.5)
   v
 DEAD    client: Pod State with power < 1, then Pod Died
   |     server: Kill Notice to each payee; Destroy Entity to every client,
   |     the dying one included; then Set Bounty 0 (§9.2.9)
   |     client: ~9 s fly-over; multiplier 1; grace re-armed
   v
 client Drop Request → DROPPING    (never release before it asks)
   ⋮
 LEAVE   Remove Entity to every other client (§7.6)
```

## 8. Message catalog

### 8.1 How to read an entry

Each message has one entry, in the order of its code. An entry gives:

- **Direction** and whether the message carries a **tag** (§4.2).
- **Layout**: the fields after the type byte (and tag, and subtype), with their encodings.
- **Fields**: the meaning of each field.
- **On receipt**: what the client does with it (S→C), or what it means (C→S).
- **Audience** (S→C only): which clients the server sends it to. The client draws whatever it is sent and never filters by recipient, so the audience is part of the protocol. Appendix B summarizes it.
- **Grade**, unless given per row.

Messages that share a type letter but differ by direction have separate entries under one heading.

The client's dispatch table has 35 slots, one for each letter from `A` to `c`. Twenty-four are used; the letters `V` to `` ` `` are ignored. *Confirmed.*

### 8.2 Index

| code | S→C | C→S | § |
|---|---|---|---|
| `A` | Release Pod | Drop Request | 8.3 |
| `B` | Start Session | — | 8.4 |
| `C` | Quit | Quit Request | 8.5 |
| `D S` | Score Status | Score Status | 8.6 |
| `D B` | Bay Status | Bay Status | 8.6 |
| `E A` | Air Support Reply | Air Support Request | 8.7 |
| `E B` | Set Bounty | — | 8.7 |
| `E C` | Muncher Attack | Effect Request (Call Muncher) | 8.7 |
| `E D H I L M R S W` | — | Effect Request | 8.7 |
| `E G` | Grenade | Grenade | 8.7 |
| `E K` | Kill Notice | — | 8.7 |
| `E N` | Ping Reply | Ping | 8.7 |
| `E P` | Module Order Reply | Module Order | 8.7 |
| `E c` | Echo Probe | Echo Reply | 8.7 |
| `E d` | — | Drone Report | 8.7 |
| `F` | Spawn Mine | — | 8.8 |
| `G` | Introduce Pod | — | 8.9 |
| `H` | Remove Entity | — | 8.10 |
| `I` | Damage | Hit Report | 8.11 |
| `J` | Destroy Entity | Pod Died | 8.12 |
| `K` | Move Vehicle | Leg Arrived | 8.13 |
| `L` | Warp Vehicle | — | 8.14 |
| `M` | Spawn Tower | Tower Order | 8.15 |
| `N` | Spawn Sentinel | — | 8.16 |
| `O` | Tower Order Reply | — | 8.17 |
| `P` | Spawn Module | — | 8.18 |
| `Q` | Module Operation | Module Operation | 8.19 |
| `R` | Ambient | — | 8.20 |
| `S` | Pod Flags | Pod Flags | 8.21 |
| `T` | Block Classes | — | 8.22 |
| `U` | Text | Chat | 8.23 |
| `a` | Text | — | 8.23 |
| `b` | Pod State | Pod State | 8.24 |
| `c` | Shot | Shot | 8.25 |
| `d` | — | Acknowledge | 8.26 |

### 8.3 `A` — Release Pod and Drop Request

#### S→C — Release Pod

**Tag:** yes.

```
id:b10
```

**On receipt.** If pod `id` is in drop state, the client releases it and it begins to fall. Otherwise the message does nothing, so repeating it is harmless. *Confirmed.*

Release Pod is needed in three places:

- at session start, for the client's own pod (§7.1);
- after every Introduce Pod, for the introduced pod (§7.2);
- after a death, in answer to the Drop Request (§7.5).

**Audience:** every client holding the pod. Each client's copy of a pod waits in drop state separately.

#### C→S — Drop Request

No fields. "My fly-over is over; drop me." Sent only after a death, never at session start (§7.5). *Confirmed.*

### 8.4 `B` — Start Session

**Direction:** S→C. **Tag:** yes.

```
self_id:b10  slot:b10  callsign:str  network:str  city:str  max_mult:b10  tower_output:b10  max_module_level:b10  n4:b10  n5:b10
```

| field | meaning | grade |
|---|---|---|
| `self_id` | the net id of the client's own pod | confirmed |
| `slot` | `(index << 3) \| team` (§6.3) | confirmed |
| `callsign` | the player's own callsign. The roster panel shows 12 characters. Mixed case is shown as sent | confirmed |
| `network` | network name, shown in the banner | confirmed |
| `city` | city name, shown in the banner | confirmed |
| `max_mult` | the city's maximum multiplier; at most 51 (§9.2.6) | established |
| `tower_output` | 0 low, 1 normal, 2 high | established |
| `max_module_level` | 0–5: the highest module level the purchase menu offers | established |
| `n4` | not read by the client; send 0 | established |
| `n5` | the capacity of a list nothing visible uses, plus 2; 2 or less means none. Send 0 | inferred |

**On receipt.** The client sets up its own pod and the city settings, and displays `City: <city> on <network>`. *Confirmed.*

Only the first Start Session of a session takes effect; later ones do nothing except reset the client's no-traffic watchdog, as any message does. The server sends it at once after launch and MAY repeat it until the client responds, within the limit in §7.1.

**Audience:** that client only.

**Example:** see Appendix A.1.

### 8.5 `C` — Quit and Quit Request

#### S→C — Quit

**Tag:** yes. No other fields.

**On receipt.** The game engine exits and the client returns to the Chamber on the same connection (§5.5). Sent in answer to a Quit Request, or unprompted to end a player's game. *Confirmed.*

**Audience:** that client only.

#### C→S — Quit Request

No fields. The player pressed Shift-Q. *Confirmed.* It follows the client's final status reports (§9.2.10, *inferred*).

### 8.6 `D` — Status

Progression state of the local player (§9.2). Both subtypes have the same layout in both directions. When the server sends one, every field overwrites the client's value, so all fields MUST carry current values.

**Tag:** S→C yes, C→S no.

#### `D S` — Score Status

```
'S'  score:num  multiplier:b10  bay_points:num
```

| field | meaning |
|---|---|
| `score` | SCORE |
| `multiplier` | score multiplier, 1 to the city maximum |
| `bay_points` | bay points |

**On receipt** (S→C): the client sets all three values and restarts its multiplier countdown at `multiplier × 30` ticks. *Established.*

**Sent by the client** every 60 ticks and on each multiplier step, once its scoring has started (§9.2.3). *Confirmed.* Also when quitting (§9.2.10, *inferred*).

**Audience:** that player's client only.

**Example** (constructed): Score Status from the server setting SCORE 0, multiplier 1, bay points 0, with tag `12`:

```
43 53 85 7a 1b  44  12  53  10  11  10  0d
                D   tag S   0   1   0
```

#### `D B` — Bay Status

```
'B'  level0:b10 … level8:b10  marker:b88
```

| field | meaning |
|---|---|
| `level0`–`level8` | the nine bay levels: 0 = no bay, 1–5 = the highest module level the bay accepts |
| `marker` | the battle marker (§9.2.7) |

**On receipt** (S→C): the client sets the bays and the marker, and starts its scoring clock if it has not already (§9.2.3). *Established.*

**Sent by the client** whenever a bay changes level (*established*), and when quitting (*inferred*).

A new player's bays are `1 1 1 1 0 0 0 0 0`, marker 0. The manual names a bay level "A"; it has no known wire value, and a server that stores and returns the client's own reports never needs one.

**Audience:** that player's client only.

### 8.7 `E` — Extended messages

A family of small messages. The subtype letter follows the tag.

**Tag:** S→C yes, C→S no.

#### `E A` — Air Support

| direction | layout | meaning | grade |
|---|---|---|---|
| C→S | `'A' priority:b10 cell x:num z:num` | the player used an Air Support module, targeting this point. `priority`: the player's Priority module is on (§9.4) | confirmed |
| S→C | `'A' code:num` | reply; 0 displays "GUPPY: Rgr, Air Support. CLEAR AREA!" | confirmed |

#### `E B` — Set Bounty

**Direction:** S→C.

```
'B'  id:b10  bounty:num
```

| field | meaning |
|---|---|
| `id` | the pod |
| `bounty` | the bounty, in units of 10: the client displays and pays 10 × this value |

**On receipt.** The client sets the pod's bounty. *Confirmed.* After the pod is created, this is the only message that changes a bounty; the client otherwise only adds to it (§9.2.4). *Established.* Values above 9 999 are displayed abbreviated in thousands.

**Units.** Elsewhere in this document a bounty is given as the client displays and pays it, 10 × the wire value, unless it says "on the wire".

**Audience:** every client.

#### `E C` — Muncher Attack

**Direction:** S→C.

```
'C'  muncher:num  target:num
```

**On receipt.** The Muncher descends onto the target, 30 ft above it, and stays latched there. The client does no damage; eating the target is the server's decision (§9.5.2). *Established.*

**Audience:** only clients holding the Muncher's create (§7.3).

In the C→S direction, `E C` with no fields is the Effect Request for Call Muncher (below).

#### Effect Requests

**Direction:** C→S. No fields after the subtype.

| code | module used |
|---|---|
| `E C` | Call Muncher |
| `E D` | Mine Sweep |
| `E H` | Harden |
| `E I` | Immunize |
| `E L` | Liberator |
| `E M` | Mine |
| `E R` | Repair |
| `E S` | Call Sentinel |
| `E W` | Weather |

The player used a one-shot module. The client displays its own message and changes nothing else; the effect is the server's to carry out, at the sender's last reported position (§9.3.5). *Confirmed.* A Module Operation used-up report follows (§8.19).

#### `E G` — Grenade

**Direction:** both; same layout.

```
'G'  cell  x:num  y:num  z:num  yaw:num  pitch:num
```

A Grenade module was fired from this point and direction. The server relays it unchanged to every other client. *Confirmed.*

**On receipt** (S→C): the client flies its own copy of the grenade and applies its blast to its own pod (§9.3.6). No reply.

**Audience:** every client except the thrower.

#### `E K` — Kill Notice

**Direction:** S→C.

```
'K'  verdict:char
```

| `verdict` | the client displays | effect |
|---|---|---|
| `E` | `** Earned Kill Recorded **` | battle marker +1 |
| any other byte | `** Kill Recorded **` | none |

Tells a player who was paid for a kill whether the kill was earned (§9.2.8). This is the only way the battle marker increases. *Confirmed.*

**Audience:** each payee.

#### `E N` — Ping

**Direction:** both. No fields after the subtype.

The client sends a Ping about every 4 s and times the reply, displaying the round trip on a latency meter. *Confirmed.* **The server MUST reply with Ping Reply at once**, not from a queue: any delay the server adds is measured and displayed. The meter draws nothing while it has no measurement, so a server that ignores pings leaves it blank. *Established.* The meter counts whole seconds of the DOS time of day, so a prompt reply reads 0 or 1 depending on where the round trip falls against a second boundary. *Established.*

**Audience** (reply): the client that pinged.

#### `E P` — Module Order

| direction | layout | meaning |
|---|---|---|
| C→S | `'P' module:b10 level:b10 priority:b10` | order a module. `module` is its catalog index (§9.3.1); `priority`: the player's Priority module is on |
| S→C | `'P' module:b10 ok:b10` | reply |

**On receipt** (S→C): if `ok` is not 0, the client displays "GUPPY: Rgr, <name> module drop."; if 0, "GUPPY: Unable, no <name> modules in stock." Either reply re-opens the purchase menu at once; without one it re-opens itself after a second. *Confirmed.*

An order costs the player nothing, so a refusal needs no refund. The module itself is delivered separately (§9.4).

**Example** (captured, C→S payload): `45 50 11 11 10`: module 1 (High Speed), level 1, no priority.

**Audience** (reply): the orderer.

#### `E c` — Echo Probe

| direction | layout | meaning | grade |
|---|---|---|---|
| S→C | `'c' value:b10` | a probe | confirmed |
| C→S | `'c' value:b10 word:num` | the reply: `value` returned, then a number (0 in every reply seen) | inferred |

Optional; a server need never send it.

#### `E d` — Drone Report

**Direction:** C→S.

| layout | meaning |
|---|---|
| `'d' 'O'` | the player turned the Drone view off |
| `'d' 'M' cell x:num z:num` | the drone's position while the view is on |

Information only; no reply and no relay. *Confirmed.*

#### Unused

The client contains `E V` and `E v` messages but never sends them.

### 8.8 `F` — Spawn Mine

**Direction:** S→C. **Tag:** yes.

```
id:num  variant:b10  cell  x:num  y:num  z:num
```

| field | meaning |
|---|---|
| `id` | the mine's net id |
| `variant` | 0–3: the mine's model; send the owning team |
| `cell x y z` | position |

**On receipt.** The client creates a proximity mine. *Confirmed.*

**Audience:** every client.

### 8.9 `G` — Introduce Pod

**Direction:** S→C. **Tag:** yes.

```
id:b10  callsign:str  slot:b10  cell  x:num  y:num  z:num  orient:num  v1:b88  v2:num  bounty:num
```

| field | meaning | grade |
|---|---|---|
| `id` | the pod's net id | confirmed |
| `callsign` | the player's callsign (§8.4) | confirmed |
| `slot` | `(index << 3) \| team` (§6.3) | confirmed |
| `cell x y z orient` | position and heading: the pod's last reported Pod State | confirmed |
| `v1`, `v2` | meaning unknown; send 0 and 0 | unknown |
| `bounty` | initial bounty, in units of 10 (as Set Bounty); 0 for none | confirmed |

**On receipt.** The client creates the pod, **in drop state**, with no check for an existing pod of the same id. The server MUST therefore:

- send it once per pod per client;
- follow it at once with Release Pod for the same id (§7.2).

*Confirmed.*

**Always include `bounty`.** The client checks whether the field is present, and would read a frame terminator as a bounty. *Established.*

**Audience:** every client except the pod's owner.

**Example:** see Appendix A.2.

### 8.10 `H` — Remove Entity

**Direction:** S→C. **Tag:** yes.

```
id:b10
```

**On receipt.** The client deletes the entity outright, with no explosion. Used when a player leaves (§7.6). *Confirmed.*

The id is a `b10` byte, so Remove Entity can name only ids 0–239. Munchers and other high-id entities are removed with Destroy Entity.

The entity disappears, but its id is taken to stay in the client's entity list (§6.2), so the id is not free again for that client: a player who leaves and comes back while the other clients are still connected MUST be given a fresh pod id. *Inferred*: how the client handles the list entry is only partly known, and a fresh id is safe either way.

**Audience:** every client except the leaving player.

### 8.11 `I` — Damage and Hit Report

#### S→C — Damage

**Tag:** yes.

```
attacker:num  damage:b10
```

| field | meaning |
|---|---|
| `attacker` | ignored by the client; send the shooter's id |
| `damage` | `magnitude × 10 + class` |

**On receipt.** The client applies damage to its own pod: a base value for the weapon class, scaled by `magnitude`. Nothing happens during spawn grace (§9.1.5). *Confirmed.*

| class | 0 | 1 | 2 | 3 | 4 |
|---|---|---|---|---|---|
| base damage | 150 | 150 | 300 | 450 | 500 |

Classes 5–9 MUST NOT be sent. *Established.* Measured damage follows the base: about 137, 345 and 481 for classes 0, 2 and 3. *Confirmed.* How `magnitude` scales the base is *inferred*. The starting gun is class 0.

**Audience:** the damaged pod's client only.

#### C→S — Hit Report

```
target:num  damage:b10
```

A projectile simulated by the sender struck `target`. `damage` uses the encoding above. The message does not say who fired; see §9.1.2. A client never sends a Hit Report naming its own pod. *Confirmed.*

### 8.12 `J` — Destroy Entity and Pod Died

#### S→C — Destroy Entity

**Tag:** yes.

```
id:num
```

**On receipt.** The entity explodes and is removed; for a pod, this is its death (§7.5). The explosion throws debris that can damage nearby pods (§9.1.4). *Confirmed.*

Used for a pod's death, and to destroy towers, mines, Sentinels and Munchers. Not for a player leaving: see Remove Entity.

**Audience:** every client, **including the destroyed pod's own client.**

#### C→S — Pod Died

No fields. The client's own pod's power fell below 1. Sent immediately after a final Pod State (§7.5). *Confirmed.*

### 8.13 `K` — Move Vehicle and Leg Arrived

Guppies (§9.4) and Munchers (§9.5.2) are both **vehicles**, and share these messages. The id selects the kind: below 260 a guppy, 260 and above a Muncher.

#### S→C — Move Vehicle

**Tag:** yes.

```
id:num  cell  x:num  z:num
```

**On receipt.** *Confirmed.*

- **An unknown id creates a vehicle at the point:**
  - a guppy flies at `220 + 10 × (id − 250)` ft, so guppy ids are 250–259; a lower id flies underground;
  - a Muncher is created 325 ft up, and heads at once for the city origin, so its create MUST be followed immediately by a second Move Vehicle giving its real destination. The two MUST arrive in that order: a destination that arrives first is itself taken as the create, at the wrong point.
- **A known id sends the vehicle to the point.** A guppy arriving there is reported; a Muncher hovers at 325 ft on arrival and is reported.

A guppy destination replaced before it is reached is never reached and never reported.

**Audience:** every client for guppies; only clients holding the create for Munchers (§7.3).

#### C→S — Leg Arrived

```
id:num
```

The vehicle reached its destination. *Confirmed.*

- Every client simulating the vehicle sends one report per leg it completes, in order, so a server receives each arrival once from each client.
- A guppy's create counts as a leg and is reported at once.
- A Muncher's create is never reported; its first report is its first dispatch's arrival.

### 8.14 `L` — Warp Vehicle

**Direction:** S→C. **Tag:** yes.

```
id:num  cell  x:num  z:num
```

**On receipt.** *Confirmed.*

- A guppy moves to the point at once and reports arrival immediately. Five warps in a row display `GUPPY MALFUNCTION`.
- A Muncher's destination is set to the point.
- An unknown id displays `ERROR: PF<id>`.

**Audience:** every client holding the vehicle.

### 8.15 `M` — Spawn Tower and Tower Order

#### S→C — Spawn Tower

**Tag:** yes.

```
carrier:num  id:num  team:b10  cell  x:num  y:num  z:num  mega:b10
```

| field | meaning |
|---|---|
| `carrier` | 0 normally. If not 0, it names an existing entity, and the tower appears 30 ft below that entity instead of at `y` |
| `id` | the tower's net id |
| `team` | the owning team |
| `cell x y z` | position |
| `mega` | 1 for a MegaTower, 0 otherwise. Only the value 1 selects a MegaTower |

**On receipt.** The client creates the tower above the point and lets it fall to the first surface below: a roof or bridge, not necessarily the ground. If `y` is below 300 the tower starts at 300 ft, otherwise 100 ft above `y`. *Confirmed.* If an entity with that id already exists, the message is ignored. *Established.*

Once a tower lands inside the city, it counts toward its team's control of its block (§9.2.6).

**Audience:** every client.

#### C→S — Tower Order

```
cell  x:num  y:num  z:num  mega:b10  priority:b10
```

The player ordered a tower (or, with `mega` 1, a MegaTower) at the pod's current position. The client has already spent one multiplier point for a tower, or two for a MegaTower. `priority`: the player's Priority module is on. *Confirmed.*

The server answers with Tower Order Reply (§8.17) and delivers the tower (§9.4).

**Example** (captured payload): `4d 10 f2 1e 32 14 32 26 10 10`: cell (0, 0), x 224, y 62, z 332, not mega, no priority.

### 8.16 `N` — Spawn Sentinel

**Direction:** S→C. **Tag:** yes.

```
id:num  cell  x:num  z:num
```

**On receipt.** The client creates a Sentinel at 300 ft above the point; it settles just above the ground. There is no altitude field. *Confirmed.* Sentinels are described in §9.5.1.

**Audience:** every client.

### 8.17 `O` — Tower Order Reply

**Direction:** S→C. **Tag:** yes.

```
result:b10
```

| `result` | the client displays | multiplier point |
|---|---|---|
| 0 | GUPPY: Rgr, relay tower drop. | kept |
| 1 | GUPPY: Unable, bad drop point. | refunded |
| 2 | GUPPY: Unable, mission queue full. | refunded |
| 3 | GUPPY: Unable, no relay towers. | refunded |
| 4 | GUPPY: Unable, power saturation! | refunded |

Every refusal refunds one multiplier point, so a refused tower costs nothing and a refused MegaTower still costs one of its two. A refusal also shortens the multiplier's countdown to at most 30 ticks. *Confirmed.*

A server MUST NOT send Tower Order Reply in answer to a Module Order: the refund would give the player a point they never spent.

**Audience:** the orderer.

### 8.18 `P` — Spawn Module

**Direction:** S→C. **Tag:** yes.

```
id:num  module:b10  cell  x:num  y:num  z:num  vx:num  vy:num  vz:num
```

| field | meaning | grade |
|---|---|---|
| `id` | the dropped module's net id | confirmed |
| `module` | catalog index (§9.3.1) | confirmed |
| `cell x y z` | starting position | confirmed |
| `vx vy vz` | starting velocity; units unknown | inferred |

**On receipt.** The client creates a module lying loose in the world. It falls and bounces, and when it comes to rest the client reports its position with a Module Operation (§8.19). *Inferred.* If an entity with that id already exists, the message is ignored. *Established.*

**Audience:** every client.

### 8.19 `Q` — Module Operation

A family of messages for picking up, installing, ejecting and using modules, distinguished by a numeric subtype. The flows are in §9.3.

**Tag:** S→C yes, C→S no.

#### S→C

```
sub:b10  value:num
```

| `sub` | meaning | grade |
|---|---|---|
| 0 | grant pickup of loose module `value`: the client installs it, or destroys it if no bay fits | confirmed; established when no bay fits |
| 5 | install module `value` (a catalog index) directly in a bay | confirmed |
| any other (2, for example) | destroy loose module `value` | confirmed for 2; established for other values |

**Audience:** pickup grant and install: that client only. Destroy: every other client (§9.3.3).

#### C→S

| `sub` | layout | meaning |
|---|---|---|
| 0 | `0 object:num` | the player asks to pick up a loose module within 50 ft |
| 1 | `1 module:num` | the player ejected a module; the bay is already empty |
| 3 | `3 object:num cell x:num y:num z:num` | a loose module came to rest here |
| 4 | `4 module:num` | an installed module was used up or destroyed and has been removed |

*Confirmed.* None needs a reply except pickup and eject (§9.3.3).

**Example** (captured payload): `51 10 32 12`: pickup request for object 32.

### 8.20 `R` — Ambient

**Direction:** S→C. **Tag:** yes.

```
sub:b10  arg:b10
```

| `sub` | meaning | `arg` | grade |
|---|---|---|---|
| 0 | set the light palette now | 0–4 | confirmed |
| 1 | cross-fade to a light palette, over about 50 s | 0–4 | confirmed |
| 2 | set the rain level now: overlay, splashes and sound | 0–3 | confirmed |
| 3 | ramp the rain to a level, one step every 15 s; a subtype 2 cancels the ramp | 0–3 | established |
| 4 | lightning off or on | 0–1 | confirmed |

**The client does not check the palette, and a value above 4 crashes it.** A server MUST NOT send a palette above 4, or a rain level above 3 (untested).

A cross-fade back to the starting palette, sent while a fade is still running, is ignored. A server SHOULD space palette changes at least 50 s apart. *Confirmed.*

The five palettes, described by their appearance (the names are not the game's):

| palette | appearance |
|---|---|
| 0 | bright daylight (the default) |
| 1 | very dark; unlit faces black |
| 2 | dim and saturated |
| 3 | dim |
| 4 | bright and washed out |

The palettes are not a day–night cycle. *Established.*

**Audience:** every client in the city, and the current state to each client that joins (§7.3).

### 8.21 `S` — Pod Flags

| direction | layout | meaning | grade |
|---|---|---|---|
| S→C | `id:b10 flags:b10` | set a pod's flag byte. Bit 3 (`flags` = 8) mounts an automatic gun on the pod (§9.1.3) | confirmed |
| C→S | `flags:b10` | the client's own pod's flags changed, or its pod was created: in bursts of 4 at session start and 2 after a death | confirmed |

**Tag:** S→C yes.

The client's reports need not be relayed; nothing in a basic game depends on them.

**Audience** (S→C): every client.

### 8.22 `T` — Block Classes

**Direction:** S→C. **Tag:** yes.

Sets the **class** of city blocks; a block's class sets its territory value (§9.2.6). Two forms, distinguished by the byte after the tag:

```
'I'  class:b10 × 64                 the whole grid
sub:char  cell:b10  class:b10       one block (sub any letter but 'I'; `C` is the usual choice)
```

- **Whole grid:** the 64 classes are in world order; byte `8 × X + Z` is block (X, Z).
- **One block:** `cell` is `(Z << 4) | X` (offset by `0x10`), **the transpose** of the cell byte in every position field.

*Confirmed.*

**Classes MUST be 0–3.** The client looks up the value of other classes without a range check and produces nonsense scores. *Confirmed.*

The client's strategic map draws block (X, Z) at column Z, row X.

**Audience:** every client, and the whole grid to each client that joins.

### 8.23 `U` and `a` — Text and Chat

#### S→C — Text

`U` (with a tag) or `a` (without).

```
sender:b10  style:b10  text:str
```

**On receipt.** The client prints a line on its heads-up display:

| `sender` | `style` | printed as | grade |
|---|---|---|---|
| 100 | any | `- text` (a system line) | confirmed |
| a pod id | 0 | `CALLSIGN> text` | confirmed |
| a pod id | not 0 | `CALLSIGN: text` | established |
| an id with no named pod | any | nothing | established |

`text` MUST contain only printable ASCII and is at most 230 bytes. *Confirmed.* Text is the server's only way to put text on the game screen.

**Audience:** for relayed chat, every client in the city, **including the sender**; for style 1, the sender's team only.

#### C→S — Chat

```
style:b10  text:str
```

The player typed a line: style 0 to everyone (the manual's `'` key), style 1 to the team (backtick). The text has no callsign. *Confirmed.*

**The client does not display its own line** until the server sends it back. The server relays it as Text with `sender` = the sender's pod id and the same style.

### 8.24 `b` — Pod State

The position and motion of a pod. The client reports its own pod about twice a second while moving and once a second when still; the server sends each client the state of the other pods.

#### S→C

**Tag:** no.

```
id:b10  cell  x:num  y:num  z:num  orient:num  fwd:b88  fwd_target:b88  v1:num  v2:num  state:b10  weight:b10
```

| field | meaning |
|---|---|
| `id` | the pod |
| `cell x y z` | position (§6.1) |
| `orient` | heading, tenths of a degree |
| `fwd` | current forward speed, ft/s |
| `fwd_target` | commanded forward speed |
| `v1`, `v2` | vertical speed (current, target) if `state` is 1; otherwise turn rate (current, target) |
| `state` | 0 on the ground, 1 falling; see below |
| `weight` | total weight of installed modules (§9.3.1) |

**On receipt.** The client moves its copy of the pod and gives it the stated speed, less 10% (below), so that its motion is smooth between updates. *Confirmed.*

- The client applies `fwd` only when `state` is 0, and treats `v1`/`v2` as vertical speed only when `state` is 1. *Established.*
- If `state` is 4, the client discards the whole message. *Established.*
- If the pod is in drop state, the client discards the message (§7.4).
- If `id` is the receiving client's own pod and `fwd_target` is 0, the client moves its pod to the given position at once and carries on from there. This is the only known way to reposition a player. *Confirmed.*

**Relaying.** When the server forwards a client's Pod State, it sends `state` as 1 if the reported state was 1, and 0 otherwise, including when the reporter is itself still in drop state. `v1` and `v2` are forwarded as reported (apart from the optional scaling below): in the client's own reports they are vertical speeds only when the reported state is 1, and turn rates in states 0, 2 and 4, so they always match the relayed state. *Established.* Forwarding `v1`/`v2` under the wrong state makes the pod spin.

**Speeds are scaled on receipt.** In a message with `state` 0, the client multiplies `fwd`, `fwd_target`, `v1` and `v2` by 0.9 (`0xe666 / 0x10000`, 16.16 fixed point), rounding down: a received value `w` becomes ⌊`w` × `0xe666` / `0x10000`⌋. Vertical speeds (`state` 1) are used unchanged. *Established.*

A server that relays these values unchanged therefore shows other players' pods walking and turning 10% slower than reported between updates, with each update moving them forward to catch up. This is probably intended. Only motion the player controls is damped, and the ballistic fall is not. That points to deliberate under-prediction: when a player stops or turns, the remote copy has not overshot, and the correction is a step forward rather than a pull back. That mattered over a modem. The original server presumably relayed the values unchanged. *Inferred.*

A server MAY undo the scaling by relaying each of the four values as `w` = ⌈`v` × `0x10000` / `0xe666`⌉, which the client turns back into exactly `v`. This must be a true ceiling, rounding toward +∞ for negative `v` too. `round(v / 0.9)` comes out 1 short for about half of all values, and in languages whose integer division truncates, `-((-a) / b)` is not a ceiling. Clamp `fwd` and `fwd_target` to the `b88` range after rounding, so reported speeds of 108–119 ft/s arrive as 107. The client keeps only the low 16 bits of a received turn rate, so clamp turn rates to −32768..32767; above about ±29,490 they cannot be reproduced exactly. The cost is overshoot: a remote pod that stops suddenly runs past the stop and is pulled back at the next update. Nobody has compared the two in play. *Server-defined.*

**Audience:** every client holding the pod (§7.2), except its owner. What a client does with a Pod State naming a pod it has never been introduced to is unknown, so a server sends none.

#### C→S

```
cell  x:num  y:num  z:num  orient:num  fwd:b88  fwd_target:b88  v1:num  v2:num  state:b10  power:num  weight:b10  armor0:num  armor1:num
```

Fields as above, without `id` (the pod is always the sender's own), and with:

| field | meaning |
|---|---|
| `state` | 4 waiting to drop, 1 falling, 2 touching down, 0 on the ground (§7.4) |
| `power` | power remaining, up to 3000; the final report before a death can be slightly negative |
| `weight` | as above, plus 100 while a High Jump or Performance module is active |
| `armor0`, `armor1` | armor remaining in the two zones |

*Confirmed.*

**Example:** see Appendix A.1.

### 8.25 `c` — Shot

A projectile or grenade launched by an entity. The message has the same layout in both directions; the server relays a client's Shot to the other clients unchanged.

**Tag:** no.

```
shooter:num  cell  x:num  y:num  z:num  yaw:num  pitch:num  kind:b10
```

| field | meaning |
|---|---|
| `shooter` | the entity that fired: the sender's own pod, or an NPC the sender is simulating (§9.1.3) |
| `cell x y z` | launch point |
| `yaw`, `pitch` | direction, tenths of a degree |
| `kind` | the weapon; see below |

**`kind`.** A value above 9 means three projectiles instead of one: subtract 10 to get the weapon. *Established.*

| kind | object | appearance | grade |
|---|---|---|---|
| 1 | projectile | yellow | confirmed |
| 2 | projectile | red-orange, easy to miss | confirmed |
| 3 | projectile | white-blue | confirmed |
| 9 | grenade | immediate explosion | confirmed |
| 11, 12, 13 | three projectiles | as 1, 2, 3, spread in three directions | confirmed |
| 0, 4–8, 10, 14–18 | — | **the client crashes** | confirmed for 0; inferred for the rest |
| 19 and above | — | never seen; untested | unknown |

`kind` names the weapon, not who fired it: a player's gun and a pod-mounted automatic gun both fire kind 1, and Sentinels fire kind 2.

Kind 9 is not how the Grenade module is thrown: a thrown grenade has its own message, Grenade (`E G`, §8.7), with its own flight and blast (§9.3.6).

A server MUST NOT send a Shot whose `kind` is outside 1, 2, 3, 9, 11, 12, 13. Shots the server creates must be checked. A server MAY drop a client's Shot with any other `kind`.

**Audience** (relayed): every client except the sender.

**Example:** see Appendix A.2.

### 8.26 `d` — Acknowledge

**Direction:** C→S.

```
tag × n
```

The tags of tagged messages the client has received, as raw bytes, in order. The client sends them in batches, at the end of each frame of its game loop or when the batch is full: an Acknowledge carries at most 11 tags. *Confirmed.*

A server MAY ignore Acknowledge, or use it to measure round-trip time.

**Example** (captured payload): `64 11 12 13 14 15`: acknowledges tags `11` to `15`.

## 9. Game systems

This section explains how the messages work together. Layouts are in §8 and are not repeated here.

### 9.1 Combat

#### 9.1.1 Who simulates what

Each client simulates every projectile it receives, but detects hits only for projectiles it originated itself. A projectile **originates** at a client when that client's player fires it, or when an entity the client is simulating fires it: an automatic gun mounted on the player's pod, or a Sentinel (§9.5.1). *Confirmed.*

So for any one projectile, exactly one client decides whether it hit anything.

#### 9.1.2 Hit reports

When a projectile a client originated strikes an entity **other than that client's own pod**, the client sends a Hit Report (`I`) naming the target. *Confirmed.*

- A Hit Report has no author field. Its author is the client whose connection it arrived on.
- One report is sent per hit, not per shot. A player's own shots that hit are almost always reported; most shots fired by a simulated NPC hit nothing and produce no report.
- A report arrives about one second after the Shot that caused it.
- No two clients report the same hit, so reports never need de-duplication.

**Crediting a hit.** A Hit Report does not say whether the sender's player fired the projectile or the sender merely simulated an NPC's shot. The server must decide from the Shots that connection sent: a hit that follows the player's own Shot is the player's, and one that follows only an NPC's Shot is credited to nobody. How the two are matched is server-defined.

#### 9.1.3 Shots not fired by a player

Three things fire without player input, and they report their shots and hits with the ordinary Shot and Hit Report messages: *established*

| firing entity | `shooter` in its Shot | credited to |
|---|---|---|
| an automatic gun on a pod (Pod Flags bit 3) | the pod carrying it | the pod's player; indistinguishable from manual fire |
| a Sentinel | the Sentinel's own id | nobody |
| a projectile object | its parent | as its parent |

A Shot or Hit Report is therefore not proof that a player pulled a trigger, and MUST NOT be treated as one for scoring, ammunition or cheat detection.

Which client simulates a given NPC is decided by the clients themselves, consistently, so the server needs no rule for it (§9.5). The server relays NPC Shots like any other.

#### 9.1.4 Damage

**To other players' pods.** A pod's own client never simulates a projectile that another client fired, so it never detects that hit. The server makes the damage happen by sending the target's client a Damage (`I`) message. Relaying the reported `damage` byte unchanged reproduces what the shooter's client resolved; what to send is server-defined (§10). *Confirmed.*

**To the reporter's own pod.** A client never reports a hit on its own pod. It applies the damage itself, at once, and tells nobody; the server sees it only as a drop in `power` in later Pod State reports. This is how NPC fire and grenades hurt players. *Confirmed.* The two paths never overlap: a client applies itself exactly the hits it does not report. A server MUST NOT send a client Damage for a hit on its own pod; that would apply it twice.

**Explosions.** When a pod or tower is destroyed, nearby pods take damage from flying debris, applied by each pod's own client with no message involved. The amount varies with distance and at random, up to about 1750 at close range and still several hundred at 400 ft. *Confirmed for a pod's destruction; established for a tower's.*

**Teams.** No damage path checks teams; friendly fire hurts. *Established.* (An automatic gun avoids aiming at teammates, but its projectiles still damage whatever they hit.)

**To structures.** The client keeps no health for towers, Sentinels or Munchers. It reports hits on them and does nothing else; when, and whether, one is destroyed is server-defined.

**Power and death.** A pod's power and armor are kept by its own client. The server cannot set them; it can only cause damage with Damage or destroy the pod with Destroy Entity (`J`). *Established.* When power falls below 1, the client sends a final Pod State and then Pod Died (`J`) (§7.5). *Confirmed.*

**Power drain.** A pod with no tower nearby loses about 14 power per second standing still and about 40 per second walking, from a full 3000. *Confirmed.*

#### 9.1.5 Spawn grace

A newly spawned pod cannot be damaged by any means, including its own client's self-applied damage. *Confirmed.*

Grace begins at every spawn and respawn, and ends at whichever comes first: *established*

- two minutes passing; or
- the pod leaving its **home box**: the 3 × 3 blocks centered on its team's home corner (§6.3). Inside the city this is the corner's 2 × 2 blocks, so a Red pod keeps grace until x or z exceeds 1200 ft.

The home box is reset whenever any pod is destroyed, to that pod's team's corner. While a player is in grace, an enemy pod's death may therefore move the player's box. *Inferred.*

The server is not told when grace ends; it can track it from the pod's reported position. A server can end a player's grace at once by moving the pod out of its home box with an own-pod Pod State (§8.24).

### 9.2 Scoring and progression

#### 9.2.1 Overview

Each player has a SCORE, a score multiplier, bay points, nine module bays and a battle marker. Each pod also carries a bounty.

While a session runs, the client computes most of these for its own player. It steps the multiplier, adds territory income, upgrades bays and credits itself for kills. The client does **not**:

- persist anything between sessions;
- know any other player's SCORE, multiplier or bays;
- judge whether a kill was *earned*;
- set a bounty to an absolute value, or ever clear one.

The server therefore supplies the starting state, every bounty value, the judgment of earned kills, and persistence. It also owns the multi-player record of who scored what.

#### 9.2.2 Quantities

*Established.*

| quantity | range | the client changes it | the server sets it with | the client reports it in |
|---|---|---|---|---|
| SCORE | 32-bit | territory income each tick (§9.2.6); kill credit (§9.2.8) | Score Status | Score Status |
| Multiplier | 1 to the city maximum | steps up on a timer (§9.2.5); reset to 1 on the local pod's death | Score Status | Score Status |
| Bay points | ≥ 0 | territory income each tick; spent on bay upgrades (§9.2.7) | Score Status | Score Status |
| Bay levels (nine) | 0–5; 0 = no bay | upgrades and downgrades (§9.2.7) | Bay Status | Bay Status |
| Battle marker | signed | −1 on the local pod's death; +1 on an earned Kill Notice; reset to 0 after a bay change | Bay Status; Kill Notice adds 1 | Bay Status |
| Bounty (every pod) | ≥ 0, in displayed units (10 × the wire value, §8.7) | +10 each tick (1 on the wire), every named pod | Set Bounty; Introduce Pod at creation | never |

#### 9.2.3 Activation

The client runs none of its per-tick scoring until it has received a Bay Status (`D B`). Until then:

- the multiplier stays at 1;
- SCORE and bay points do not accrue;
- bounties do not accrue;
- the client sends no Score Status or Bay Status reports.

*Confirmed for the multiplier; established for the rest.*

A server MUST send Bay Status at session start, before Score Status (§9.2.11).

#### 9.2.4 The tick

The client's scoring runs once per **tick** of a timer with a nominal period of one second. A tick is always somewhat longer than a second, and its length varies a little between clients and sessions. *Established; the measured rate is confirmed.*

- The client checks the timer once per frame of its main loop, against the DOS time of day. A tick fires on the first check at which more than 100 hundredths of a second have passed since the previous tick fired, and the next tick is timed from that check.
- The DOS time of day advances in steps of one BIOS timer tick (about 55 ms, 18.2 per second). So the shortest possible tick is **19 BIOS ticks, 1.0436 s**. A tick is one BIOS tick longer whenever no frame runs during the BIOS tick in which it falls due.
- Measured rates are **1.044–1.050 s** per tick, most often 1.050 s. The client's timers therefore run about 5% slow against a server's clock, and the gap grows over a session.
- While the local pod is dying (from its power running out until its Destroy Entity arrives), the timer does not run: the multiplier countdown, territory income and bounty accrual all pause. *Established.*

A server SHOULD NOT model this clock. It takes the multiplier and SCORE from the client's Score Status reports (§9.2.10).

Timers outside scoring (spawn grace, the rain ramp, the kill-credit window, the latency meter) use a separate clock that counts whole seconds of the DOS time of day and does not drift.

Each tick, once scoring is active, the client:

1. advances the multiplier countdown (§9.2.5);
2. adds territory income to SCORE and to bay points (§9.2.6);
3. upgrades the lowest bay if bay points have reached its threshold (§9.2.7);
4. adds 10 (1 on the wire) to the bounty of every named pod, including pods that are dead or waiting to drop. *Confirmed.*

#### 9.2.5 Multiplier

The multiplier starts at 1. While it is below the city maximum (Start Session's `max_mult`), a countdown decreases by one each tick. When the countdown reaches zero, the multiplier increases by 1 and the countdown restarts at (new multiplier × 30) ticks. *Confirmed.*

- The client sends Score Status on each step.
- A received Score Status sets the multiplier and restarts the countdown at (multiplier × 30).
- Spending points on a tower (§8.15) lowers the multiplier; a refused order refunds it (§8.17).
- The local pod's death resets the multiplier to 1 (§9.2.9).

#### 9.2.6 Territory income

The city has 64 **blocks**. Each block has a **class**, 0–3, set by Block Classes (`T`). All blocks start at class 0. A block's value is fixed by its class: *established*

| class | 0 | 1 | 2 | 3 |
|---|---|---|---|---|
| value | 1 | 2 | 5 | 10 |
| map outline | green | yellow | orange | red |

A team **controls** a block when it, and no other team, has at least one power tower there. Empty and contested blocks score for nobody. A tower counts from the moment it lands inside the city until its Destroy Entity. *Established.*

Each tick the client adds the local team's income to SCORE and to bay points:

> income = (sum of the values of the blocks the local player's team controls) × multiplier

*Confirmed.*

Each client computes control from the towers *it* has been sent. All clients in a city MUST therefore receive the same Spawn Tower and Destroy Entity messages and the same block classes, late joiners included (§7.3). Otherwise their territory, and so their scores, diverge.

**Overflow.** The client computes income as a 16-bit signed product. A product of 32 768 or more is added as a negative number, and SCORE decreases. A server MUST keep (city maximum multiplier) × (largest possible team income) below 32 768. With the values above, a team's income is at most 640, so any maximum multiplier up to 51 is safe. *Established.*

#### 9.2.7 Bays and progression

A pod has nine bays. A bay's level (0–5) is the highest module level it accepts (§9.3.1); level 0 means no bay. The **lowest bay** is the bay with the lowest level, empty bays included, so empty bays are filled first. *Confirmed.*

A bay is upgraded by either of two routes.

**Bay points.** When bay points reach the threshold for the lowest bay's current level, the client raises that bay one level and subtracts the threshold. *Confirmed.*

| current level | 0 | 1 | 2 | 3 | 4 |
|---|---|---|---|---|---|
| points to upgrade | 1 000 | 10 000 | 100 000 | 1 000 000 | 10 000 000 |

The client reports the upgrade with Bay Status, but sends **no** Score Status. See §9.2.11 for why this matters.

**The battle marker.** The marker rises by 1 when the client receives an earned Kill Notice (§9.2.8), and falls by 1 when the local pod is destroyed. After each change:

- if the marker ≥ the lowest bay's level, the lowest bay is upgraded;
- if −marker ≥ the highest bay's level, the highest bay is downgraded, unless the four base bays are all already at level 1.

After an upgrade or downgrade the marker returns to 0. *Established.*

The client displays `** Bay N upgraded to level L!` (or the downgrade text) and reports the change with Bay Status. Upgrades are *confirmed*. Downgrades are *established* only.

While the lowest bay is at level 1 or below, every earned kill therefore upgrades a bay.

#### 9.2.8 Kill credit

Each client credits kills to its own player only.

When the client reports a hit on a pod (Hit Report), it records the time, in whole seconds, on that pod. When it later receives a Destroy Entity for that pod, it checks two things: the recorded hit is less than 20 s old, and the pod is on another team. If both hold, the client adds (that pod's bounty × the local multiplier) to SCORE and displays:

```
Bounty Earned: B x M = P
```

*Confirmed.*

This is the manual's payout rule, evaluated by each payee for itself. It follows that:

- Every client that hit the victim within the window credits itself the **whole** bounty. It is not divided, and it does not depend on who landed the final hit, nor on whether the kill was earned.
- The bounty used is the payee's **own copy**: the last Set Bounty value plus the client's own per-tick accrual since then. It can differ from the server's figure by about 10.
- The bounty is read when Destroy Entity arrives (§9.2.9).

**Earned kills.** The client cannot tell whether a kill was earned, because it does not know other pods' bays. The manual's rule: a kill is earned when the victim's highest bay level ≥ the attacker's lowest bay level − 1. The server judges this and sends each payee a Kill Notice (`E K`): `E` for earned, any other byte for not (§8.7). An earned Kill Notice is the only way the battle marker increases. *Confirmed.*

#### 9.2.9 Death

When the local pod is destroyed, the client resets its own state and reports none of it. It:

- resets the multiplier to 1;
- lowers the battle marker by 1, which may downgrade a bay (reported with Bay Status);
- restores power to 3000;
- clears every installed module (§9.3);
- re-arms spawn grace (§9.1.5).

The server learns the new multiplier from the client's next Score Status. *Established; confirmed for the multiplier; inferred for clearing modules (§9.3.4).*

**The server's part of a death**, in wire order:

1. **Kill Notice** to each payee: every non-teammate who hit the victim in the preceding 20 seconds.
2. **Destroy Entity** naming the victim, to every client, **including the victim's own**. The victim's client does not complete its death without it. *Confirmed.*
3. **Set Bounty** naming the victim, with bounty 0, to every client. Nothing in the client clears a bounty, so a respawned pod would otherwise keep its old one. This MUST come after Destroy Entity: payees credit themselves from the bounty at the moment Destroy Entity arrives, and a zero sent first makes every payout 0. *Confirmed.*

**When the pod leaves drop state**, at the start of every life including the first, the server SHOULD send Set Bounty 0 again. Every client goes on adding to the pod's bounty through the fly-over and the drop (§9.2.4), so without the second zero, a kill early in the new life pays for time spent dead. For the first life it also corrects the bounty a joining client was given in Introduce Pod. *Established.*

Sending the Kill Notice before Destroy Entity, as above, is *confirmed* to work. Whether the client also accepts it after Destroy Entity is unknown.

**The respawn multiplier floor.** The manual says that a spawn after destruction raises a too-low multiplier to the city's maximum module level. The client does not implement this. A server that wants it sends a Score Status. Because Score Status also overwrites SCORE and bay points, it must carry values the client reported **after** the death (§9.2.10). Server-defined.

#### 9.2.10 Reports and overwrites

Once scoring is active, the client sends:

| message | contents | when | grade |
|---|---|---|---|
| Score Status | SCORE, multiplier, bay points | every 60 ticks, and on each multiplier step | confirmed |
| Bay Status | nine bay levels, battle marker | on every bay change | established |
| Score Status, Bay Status, Score Status, Quit Request | live values | on quit (Shift-Q) | inferred |

A server's Score Status overwrites all three of its values, and its Bay Status overwrites all nine bays and the marker. Neither can change one field alone. A server that sends either MUST fill every field with the player's current values.

The server's own copy of a player's SCORE and multiplier will drift from the client's: the tick is longer than a second (§9.2.4), and each client pays kills from its own bounty copy. What the client reports is what the player sees.

#### 9.2.11 Persistence

The client persists nothing. A server restores a player at session start by sending:

1. Bay Status (bays and marker), which also activates scoring (§9.2.3), then
2. Score Status (SCORE, multiplier, bay points).

*Confirmed.*

Which values carry over between games is server-defined. The manual says bays and bay points persist and the multiplier restarts each game.

Saving from the client's reports has two hazards:

- **Stale reports.** A Score Status the client built before the restore arrived carries its pre-restore values: SCORE 0 and bay points 0. Saving it erases what was restored. *Confirmed.*
- **Unreported spending.** A points-driven upgrade sends Bay Status but not Score Status (§9.2.7). Until the next Score Status, the latest points figure the server holds predates the spend. A save in that window records both the new bay level and the points that paid for it, and a restore from it awards the upgrade twice. The case is recognisable: one bay up one level, with the marker unchanged. *Confirmed.*

Bay Status reports need no such guard: the client sends them only after the server's own Bay Status has activated scoring.

The marker change on the player's own death is not reported unless it causes a downgrade. A server that saves the marker applies the −1 itself when the player dies.

#### 9.2.12 Summary of server obligations

A server MUST:

- send Bay Status at session start, then Score Status;
- set every pod's bounty with Set Bounty, and zero it after the pod's Destroy Entity;
- send every client in a city the same towers, tower destructions and block classes;
- keep (maximum multiplier × maximum team income) below 32 768;
- fill every field of any Score Status or Bay Status it sends with current values.

A server SHOULD:

- judge earned kills and send a Kill Notice to each payee;
- send Set Bounty 0 when each pod leaves drop state (§9.2.9);
- keep its own count of each living pod's bounty at the manual's rate, 10 per second (1 per second on the wire), for the bounty in Introduce Pod and its own record of payouts, and push it with Set Bounty from time to time. Every client also accrues bounties itself, slightly slower (§9.2.4); each push overwrites its count;
- keep each player's progression between sessions, if progression is meant to persist.

A server MAY apply the manual's respawn multiplier floor.

### 9.3 Modules

#### 9.3.1 The module catalog

A module is named on the wire by its **catalog index**. Indices are grouped in tens by level: the client computes a module's level as `(index − 1) ÷ 10 + 1`. So 1–9 are level 1, 11–18 level 2, 21–30 level 3, 31–37 level 4 and 41–47 level 5; other indices are unused. A bay accepts a module whose level is at most the bay's level. *Established.*

**Weight** is added to the pod's weight when a module is installed and subtracted when it is removed; the client reports the total in every Pod State. **Use** is how the module is used: one-shot modules are removed when used, toggles are switched on and off, and passive modules simply work while installed. *Index, level and weight established; use inferred.*

| index | name | level | weight | use |
|---|---|---|---|---|
| 1 | High Speed | 1 | 1 | passive |
| 2 | High Jump | 1 | 1 | passive |
| 3 | Fast Turn | 1 | 1 | passive |
| 4 | Recon | 1 | 1 | passive |
| 5 | Rocket Pack | 1 | 4 | one-shot |
| 6 | Repair | 1 | 1 | one-shot |
| 7 | Long Shot | 1 | 1 | passive |
| 8 | Infra Red | 1 | 1 | toggle |
| 9 | Resonator | 1 | 1 | passive |
| 11 | Immunize | 2 | 2 | one-shot |
| 12 | Rapid Fire | 2 | 1 | passive |
| 13 | Priority | 2 | 2 | toggle |
| 14 | Reserve Power | 2 | 2 | one-shot |
| 15 | Weather | 2 | 1 | one-shot |
| 16 | Mine | 2 | 4 | one-shot |
| 17 | Call Muncher | 2 | 2 | one-shot |
| 18 | Harden | 2 | 1 | one-shot |
| 21 | Mega Tower | 3 | 8 | one-shot |
| 22 | Front Shield | 3 | 3 | toggle |
| 23 | Stealth | 3 | 3 | passive |
| 24 | Phalanx | 3 | 4 | toggle |
| 25 | Mega Shot | 3 | 3 | toggle |
| 26 | Tri-Shot | 3 | 9 | toggle |
| 27 | Rear Shield | 3 | 3 | toggle |
| 28 | Power Enhance | 3 | 3 | passive |
| 29 | Mine Sweep | 3 | 8 | one-shot |
| 30 | Grenade | 3 | 3 | one-shot |
| 31 | Liberator | 4 | 7 | one-shot |
| 32 | Air Support | 4 | 4 | one-shot |
| 33 | Shield Enhnce | 4 | 2 | passive |
| 34 | Descrambler | 4 | 5 | passive |
| 35 | Performance | 4 | 4 | passive |
| 36 | Call Sentinel | 4 | 6 | one-shot |
| 37 | MkIII Radar | 4 | 4 | passive |
| 41 | Leech | 5 | 8 | passive |
| 42 | Cloak | 5 | 3 | toggle |
| 43 | Blaster | 5 | 6 | passive |
| 44 | Mega Cell | 5 | 4 | passive |
| 45 | Drone | 5 | 1 | toggle |
| 46 | Deflector | 5 | 5 | passive |
| 47 | Turret | 5 | 4 | passive |

The modules seen reported as used up (Module Operation subtype 4) are 15, 17, 21, 30, 31, 32 and 36, all one-shot.

#### 9.3.2 Getting modules

A module reaches a pod in one of three ways:

- **Ordered and delivered.** The player sends a Module Order (`E P`); the server replies, and delivers the module as a loose object near the player (§9.4).
- **Picked up.** See §9.3.3.
- **Installed directly** by the server with Module Operation subtype 5. The client fits the module into a bay itself; if no free bay of a suitable level exists, the module is not installed: the client holds it in a pending list, and whether it is installed later is unknown. Nothing is reported. *Established; the pending list is inferred.*

The purchase menu offers modules up to the city's maximum module level (Start Session). Which modules are in stock is server-defined; a refused order is answered with a Module Order reply with `ok` 0.

#### 9.3.3 Loose modules

*Confirmed.*

```
 Spawn Module (P) id ─► it falls and bounces; the client reports Module Operation 3 id + position at rest
 a player within 50 ft picks it up ─► client: Module Operation 0 id
   server: Module Operation 0 id to THAT client (installs it, or destroys it if no bay fits)
           Module Operation destroy id to every other client
 the player ejects a module ─► client: Module Operation 1 module (the bay is already empty)
   server: Spawn Module near the pod
```

#### 9.3.4 The installed loadout

A server can know exactly what each pod carries.

- Modules are installed only by server messages: a pickup grant or a direct install.
- Every removal is reported with the module's index: an eject (subtype 1), or used up, destroyed by fire, or a shield beaten down (subtype 4).
- A death clears every installed module. *Inferred.*

The weight in each Pod State is a check: it equals the sum of the installed modules' weights, plus 100 while a High Jump or Performance module is active. Because an install can fail silently, a server SHOULD treat each install as pending until the reported weight includes it. *Established.*

#### 9.3.5 Effect modules

Using a one-shot effect module makes the client display its own message and send an Effect Request (§8.7), then a used-up report. The client changes nothing else; the effect is the server's to carry out, at the sender's last reported position. *Confirmed.*

The effects, as the manual describes them. All are server-defined; [Part II §16.1](server-policy.md#161-effect-modules) suggests one implementation, and [Part II §16.2](server-policy.md#162-munchers) its Munchers.

| request | module | effect |
|---|---|---|
| `E R` | Repair | restores the sender's team's towers within one block to full strength |
| `E W` | Weather | changes the city's weather (Ambient, §8.20) |
| `E H` | Harden | doubles the damage capacity of the sender's team's towers within one block |
| `E L` | Liberator | destroys enemy towers within one block. The client has already taken about a quarter of the sender's power |
| `E C` | Call Muncher | brings a Muncher to the sender (§9.5.2) |
| `E S` | Call Sentinel | brings a Sentinel to the sender |
| `E I` | Immunize | makes the sender's team's towers within one block immune to Munchers for 30 minutes |
| `E M` | Mine | drops a mine at the sender that does not harm its team |
| `E D` | Mine Sweep | destroys enemy mines within one block |
| `E A` | Air Support | the team's guppy drops a grenade on the given point; the server acknowledges with Air Support Reply 0 |

#### 9.3.6 Grenades

When a player fires a Grenade module, the client sends a Grenade (`E G`) giving the launch point and direction. The server relays it unchanged to every other client. *Confirmed.*

Each client flies its own copy and applies the blast to its own pod only: within 550 ft, damage `(550 − distance) × 4`, with no team check. So without the relay a grenade can hurt only its thrower, and with it a grenade hurts anyone nearby, teammates included. *Confirmed.*

#### 9.3.7 Drone

The Drone module's view is client-side. While it is on, the client reports the drone's position with Drone Reports; these need no reply. *Confirmed.*

### 9.4 Logistics: orders and guppies

#### 9.4.1 The guppy

A **guppy** is a flying delivery vehicle, created, flown and reported with Move Vehicle, Warp Vehicle and Leg Arrived (§8.13, §8.14). *Confirmed.*

- It is drawn in the color of team `id − 250` (250 Red, 251 Blue, 252 Green, 253 Black).
- Every client simulates every guppy, so each leg is reported once by every client. The first report of a leg is its arrival; the rest are duplicates.
- **A server SHOULD NOT redirect a guppy before its current leg is reported**, because a replaced destination is never reported (§8.13). A client's n-th report is then exactly leg n, and no timing is needed.
- **This holds for each client separately.** The server acts on the first report of a leg, but another client's copy of the guppy may still be flying that leg. A server SHOULD NOT send the next Move Vehicle for a guppy to a client that has not itself reported the leg in flight: the destination replaces the leg, which that client then never reports (§8.13). Its count runs one leg behind from then on, and once the faster clients have left, nothing moves the guppy. Treat such a client as one being brought into step (§7.3): send it nothing, and answer its report with the destination now in force, or send nothing if the guppy is idle at home. *Inferred from §8.13; reproduced with simulated clients at different speeds and latencies, not yet seen with a live client.*
- Legs from a home corner to a nearby point take 14–27 s. The guppy's speed is unknown.

#### 9.4.2 Orders

*Confirmed.*

**Tower orders.** The client sends a Tower Order at the pod's position, having already spent the multiplier point (§8.15). The server answers with a Tower Order Reply: 0 accepts, 1–4 refuse and refund (§8.17).

**Module orders.** The client sends a Module Order. The server answers with a Module Order Reply: `ok` not 0 accepts, 0 refuses ("no modules in stock") and needs no refund.

**Priority.** Every Tower Order, Module Order and Air Support Request carries a `priority` flag: the orderer's Priority module is on. The manual says priority deliveries go ahead of teammates' requests.

#### 9.4.3 Delivery, as the manual describes it

*Server-defined.*

- An accepted order is flown in by the team's guppy and delivered when it arrives: a tower at the requested point (Spawn Tower), a module about 100 ft above and 20 ft ahead of where the orderer was facing (Spawn Module). A Module Order carries no position, so "where the orderer was" is its last Pod State before the order.
- A module ordered in the team's own home corner block, or the three blocks just outside the city beside it, arrives at once. For Red's corner (0, 0) those three are (−1, 0), (0, −1) and (−1, −1); the other corners are the mirror images.
- An order anywhere in the 3 × 3 blocks around an **enemy** home corner is refused: a tower with Tower Order Reply 1 ("bad drop point"), a module with Module Order Reply `ok` 0.

A typical delivery:

```
 Move Vehicle guppy (create, at its corner)  ─► Leg Arrived (create leg)
 Move Vehicle guppy dest = order point       ─► Leg Arrived (arrived)
 Spawn Tower or Spawn Module at the point
 Move Vehicle guppy dest = home              ─► Leg Arrived — next job, or wait
```

A server with no logistics MAY simply refuse every tower order (Tower Order Reply 3, "no relay towers") and every module order. [Part II §14](server-policy.md#14-logistics) suggests one implementation of the rules above, and [Part II §15](server-policy.md#15-module-stock) one stock policy.

### 9.5 NPCs

#### 9.5.1 Sentinels

A **Sentinel** is a stationary gun emplacement, created with Spawn Sentinel (`N`). *Confirmed.*

- It fires at the **globally nearest pod** if that pod is within 600 ft. The client whose own player is nearest simulates the Sentinel's fire; because only one pod can be nearest, only one client fires a given Sentinel at a time. The job passes between clients as players move. The server simply relays the Shots.
- It fires kind 2: red-orange shots that are easy to miss on screen.
- It has not been seen to distinguish teams.
- Its hits on other entities are reported like any other (§9.1.2) and credit nobody. Its hits on the simulating client's own pod are applied by that client (§9.1.4).
- The client keeps no health for it. The server destroys it with Destroy Entity, which plays an explosion and debris like a tower's. A new Sentinel can then be spawned normally.

**Automatic guns** follow a different rule:

| | Sentinel | pod-mounted automatic gun |
|---|---|---|
| fires when | the nearest pod is the simulating client's own player, within 600 ft | its owner is the simulating client's player |
| which client fires | changes as players move | always the owner's |
| distinguishes teams | not seen to | yes: skips teammates |
| targets | the nearest pod | pods only |

*Confirmed.*

#### 9.5.2 Munchers

A **Muncher** is a flying NPC that eats towers. The manual describes it as neutral, discouraged or destroyed by weapons, and often drawn to the team controlling the most blocks. *Established unless marked.*

- It is created and moved with Move Vehicle (§8.13): a create followed at once by its destination, in that order. Its create is never reported, so a server counts legs from the dispatches only. *Confirmed.*
- Muncher Attack (§8.7) latches it onto a target. *Confirmed.*
- **The client never damages anything with a Muncher.** Eating a tower, the Muncher's own durability, and its departure are all server-defined.
- It is removed with Destroy Entity (§8.12); Remove Entity cannot carry its id.
- Its messages go only to clients holding its create, and a joining client is brought in as described in §7.3. So is a client whose copy has not yet reported the leg another client's report acted on (§9.4.1).

### 9.6 Weather

The light and rain are controlled with Ambient (`R`, §8.20): a palette set at once or faded over about 50 s, a rain level set at once or ramped, and lightning. The client does nothing on its own; every change comes from the server. The current state MUST be sent to each joining client (§7.3).

---

## 10. Server obligations

The client was one half of a pair, and the original server owned these decisions. No original server, server source or recorded server traffic is known to survive. The decisions are absent from the client because the server made them, not because analysis missed them. Every server must make them. Part II suggests a way to make each.

| decision | what the client provides | constraint |
|---|---|---|
| **Identity** | a name, and a password unless the name is `JOINGENIE` (§5.1) | `JOINGENIE` carries no password; whether a name may be shared is the server's choice |
| **Callsigns** | nothing: the server sends each player's callsign (§8.4, §8.9) | separate from the account name; how it is chosen is the server's choice; at most 12 characters are shown |
| **The lobby** | a terminal, line input and buttons (§5.2–5.3) | the reply to a bare `/CITY` must be plain text |
| **Cities** | three settings in Start Session (§6.4) | maximum multiplier ≤ 51 |
| **Teams and slots** | four teams, four slots each (§6.3) | slots unique within a city |
| **Damage between players** | Hit Reports with a damage byte (§9.1.4) | the client applies the byte it is sent |
| **Hit credit** | Hit Reports without an author; Shots naming their shooter (§9.1.2) | an NPC's hit credits nobody |
| **Trust** | each client is authoritative for its own hits and survival (§11) | the server cannot set power |
| **Structure durability** | Hit Reports on towers, Sentinels and Munchers (§9.1.4) | destroy with Destroy Entity |
| **Territory values** | the class of each block (Block Classes) | classes 0–3; same grid to every client |
| **Bounty** | display and payout from the server's value (§9.2.8) | the server zeroes it after each death |
| **Earned kills and payouts** | Kill Notice (§9.2.8) | the server alone knows every pod's bays |
| **Progression across sessions** | Score Status and Bay Status, both ways (§9.2.11) | beware stale reports and unreported spending |
| **Respawn** | the client respawns at its corner when released (§7.5) | answer only the Drop Request after a death |
| **Relayed motion** | Pod State speeds and turn rates, scaled by 0.9 on receipt (§8.24) | whether to undo the scaling is the server's; use a ceiling if so |
| **NPC population** | Spawn Sentinel; Move Vehicle for Munchers (§9.5) | nothing in the client schedules NPCs |
| **Logistics and stock** | orders, replies, the guppy (§9.4) | never redirect a guppy mid-leg |
| **Module effects** | Effect Requests (§9.3.5) | applied at the sender's last position |
| **Power economy** | the tower output setting (§6.4); Spawn Tower (§8.15) | where towers may go and how much they supply are the server's |
| **Weather** | Ambient (§8.20) | palettes 0–4, rain 0–3, fades ≥ 50 s apart |
| **Match structure** | Quit to end a player's game (§8.5) | the client has no notion of a match ending |

---

## 11. Trust and security considerations

### 11.1 The client is authoritative for its own survival

A pod's power, armor, damage taken and death are all decided by its own client. A client applies damage from projectiles it simulates hitting its own pod and never reports them (§9.1.4); the server learns the result only from the `power` in that client's Pod State. A modified client could report full power forever. This is how the protocol works, not a defect of any implementation.

Equally, the client that fired a projectile is authoritative for whether it hit (§9.1.2), and could report hits that never happened.

### 11.2 No anti-cheat mechanism

The protocol contains no sequence numbers, nonces, challenges, signatures, shared secrets, timing attestation or server-issued tokens, and no way for the server to read client state other than what the client sends. *Established, for the usual forms of these mechanisms.* What exists protects only against transmission errors:

- the frame checksum (§4.1) detects corruption, not forgery;
- the tag and Acknowledge (§4.2) confirm delivery, not truthfulness.

### 11.3 What the server can enforce

The server can override almost everything:

| message | effect | grade |
|---|---|---|
| Destroy Entity | destroys any pod | confirmed |
| Remove Entity | deletes any entity outright | confirmed |
| Pod State naming the client's own pod | moves it anywhere, instantly (`fwd_target` 0, `state` 0) | confirmed |
| Damage | applies damage the client cannot refuse, outside spawn grace | confirmed |
| Pod Flags | rewrites a pod's flags | confirmed |
| Module Operation install | grants modules; every module is server-issued | confirmed |
| Score Status, Bay Status, Set Bounty | set score, multiplier, bays and bounty | confirmed |

What the server **cannot** do is set power or armor. The only routes to a pod's power are Damage (bounded per hit, and zero during grace) and Destroy Entity. *Established.*

### 11.4 What the server can check

Every Pod State from a client carries more than the server needs, and the redundancy allows plausibility checks without any change to the protocol:

| claim | can be checked against |
|---|---|
| position | elapsed time × reported forward speed |
| forward speed | actual movement between reports |
| power | the known drain rates (§9.1.4) and the positions of friendly towers |
| armor | the damage the server itself sent |
| weight | the modules the server issued (§9.3.4) |
| state | the legal sequence 4 → 1 → 2 → 0 |
| Hit Reports | the Shots from the same connection; more hits than shots is a signature |
| a Shot's `shooter` | the sender's own pod or an NPC the server created |

None of these is certain. A careful cheat can simulate a plausible pod; the checks raise the cost from editing one value to reimplementing the game's physics.

---

## Appendix A. Worked captures

All frames are from sessions with the original client. Hex is the whole frame; `·` marks where the payload starts. `tt` is a tag whose value does not matter.

### A.1 One client, session start

```
S→C Start Session
  43 53 4e b1 3b · 42 11 10 10 18 4a4f494e47454e49 15 47454e4945
                   1d 435942455243495459204f4e45 18 11 12 10 10 0d
  CK 4e  ~CK b1  LEN 3b (5 + 38)  type B  tag 11
  self_id 10 → 0   slot 10 → index 0, team 0
  callsign 18 → 8 "JOINGENI"   network 15 → 5 "GENIE"   city 1d → 13 "CYBERCITY ONE"
  max_mult 18 → 8   tower_output 11 → 1 (normal)   max_module_level 12 → 2   n4 10   n5 10   END 0d

C→S Pod Flags      43 53 8a 75 17 · 53 10
  flags 0: the client's pod exists. No terminator; the next frame follows directly

S→C Release Pod    43 53 19 e6 18 · 41 20 10 0d
  tag 20, pod 0. Repeated until the pod reports a state other than 4

C→S Pod State      43 53 25 da 28 · 62 10 b2 19 12 20 60 12 2e 88 88 10 10 14 12 d8 10 10 10
  cell 10 → (0, 0)   x b2 19 → 145   y 12 20 → 240   z 60 → 5   orient 12 2e → 450 (45°)
  fwd 88 → 0   fwd_target 88 → 0   v1 10 → 0   v2 10 → 0   state 14 → 4 (drop state)
  power 12 d8 → 3000   weight 10 → 0   armor 10 10 → 0, 0

C→S Ping           43 53 3a c5 17 · 45 4e
S→C Ping Reply     43 53 62 9d 18 · 45 27 4e 0d          tag 27; sent at once

C→S Pod State      … state 1, y 229                       falling: stop sending Release Pod

C→S Acknowledge    43 53 6e 91 1b · 64 11 12 13 14 15    tags 11–15

C→S Hit Report     43 53 69 96 1a · 49 24 31 11 10
  target 24 31 11 → 1 + 33 × 15 + 1 × 3600 = 4096 (a Sentinel)   damage 10 → class 0
```

### A.2 Two players

Two clients on the same team: P1 is pod 0 ("JOINGENI"), P2 is pod 1 ("JOINGEN2").

```
-- P2 is introduced to P1; Introduce Pod is followed at once by Release Pod
S→C Introduce Pod  43 53 34 cb 2e · 47 3a 11 18 4a4f494e47454e32 18 10 12 15 12 20 12 15 12 2e 88 10 10 0d
  tag 3a   id 11 → 1   callsign 18 → 8 "JOINGEN2"   slot 18 → index 1, team 0   cell 10 → (0, 0)
  x 12 15 → 75   y 12 20 → 240   z 12 15 → 75   orient 12 2e → 450
  v1 88 → 0   v2 10 → 0   bounty 10 → 0
S→C Release Pod    43 53 35 ca 18 · 41 3b 11 0d          pod 1
S→C Pod State      43 53 49 b6 26 · 62 11 10 12 15 12 20 12 15 12 2e 88 88 10 10 10 10 0d
  P2's state relayed: pod 1, same position, speed 0, state 0, weight 0

-- P1 throws a grenade; P2 receives the same fields with a tag
C→S Grenade        43 53 2f d0 22 · 45 47 32 b2 16 d2 14 12 24 f2 49 32 ee
  cell 32 → (2, 2)   x 100   y 72   z 300   yaw 869   pitch 3332
S→C Grenade        43 53 37 c8 23 · 45 87 47 32 b2 16 d2 14 12 24 f2 49 32 ee 0d     to P2, tag 87

-- Both die to each other's grenades. P1's view:
S→C Destroy Entity 43 53 5f a0 18 · 4a 4d 20 0d          pod 1 explodes
C→S Pod Died       43 53 70 8f 16 · 4a                   P1's own pod died
S→C Destroy Entity 43 53 51 ae 18 · 4a 4f 10 0d          pod 0, sent to P1 itself
C→S Drop Request   43 53 67 98 16 · 41                   ~10 s later: answer with Release Pod 0

-- P1 shoots P2 (payloads)
C→S Shot           · 63 10 21 d2 1d 32 1e b2 1f 22 e5 10 11
  shooter 10 → 0 (itself)   cell 21 → (1, 1)   x 207   y 212   z 235   orient 3196   pitch 0   kind 1
  relayed unchanged to every other client
C→S Hit Report     · 49 20 10                            target 20 → 1, damage class 0
S→C Damage         43 53 3c c3 19 · 49 2a 10 10 0d       to P2 only, tag 2a (constructed): attacker 0, damage class 0

-- Chat: the client shows nothing until the server's Text arrives
C→S Chat           43 53 7b 84 1a · 55 10 12 6f 6b       style 0, "ok"
S→C Text           43 53 18 e7 1b · 61 10 10 12 6f 6b 0d sender 0, style 0, "ok" → "JOINGENI> ok"

-- Quit
C→S Quit Request   43 53 69 96 16 · 43
S→C Quit           43 53 4f b0 17 · 43 65 0d             tag 65; built new, not echoed
  then "QUIT\r" twice from the client; a few seconds later 05 "A\r". Other clients get Remove Entity 10.
```

## Appendix B. Audience summary

The client draws whatever it is sent, so who receives a message is part of the protocol. A pod's own Introduce Pod duplicates it; its own Pod State sent back drags it to a stale position.

| audience | messages |
|---|---|
| **that client only** | Start Session; Quit; Score Status; Bay Status; Damage (the victim); Ping Reply; Kill Notice (each payee); Module Order Reply, Tower Order Reply, Air Support Reply (the orderer); Module Operation pickup grant and install; a Pod State naming the client's own pod |
| **every client** | Release Pod (every client holding the pod); Destroy Entity (including a dying pod's own client); Spawn Tower; Spawn Module; Spawn Mine; Spawn Sentinel; Ambient; Block Classes; server-made Pod Flags; Set Bounty; guppy Move Vehicle and Warp Vehicle; Text (including the sender); server-made Shot and Grenade |
| **every client but one** | Introduce Pod (not the pod's owner); relayed Pod State (every client holding the pod, not its owner); Remove Entity (not the leaver); relayed Shot (not the sender); relayed Grenade (not the thrower); Module Operation destroy after a pickup (not the picker) |
| **one team** | Text of style 1 |
| **clients holding the create** | Move Vehicle and Muncher Attack for a Muncher (§7.3) |

## Appendix C. Timing summary

Every timing in this specification, gathered in one place. The section given is authoritative; this table only collects them.

**Client timings.** What the client does on its own clock.

| timing | value | § |
|---|---|---|
| scoring tick | at least 1.0436 s; typically about 1.05 s; paused while the local pod is dying | 9.2.4 |
| multiplier step | every (multiplier × 30) ticks | 9.2.5 |
| Score Status report | every 60 ticks, and on each multiplier step | 9.2.10 |
| Pod State report | about 2 per second moving, 1 per second still | 8.24 |
| Ping | about every 4 s | 8.7 |
| "no session yet" CR | a few seconds after launch with no Start Session | 7.1 |
| death fly-over | about 9 s; the pod reports state 4 from about 1.2 s | 7.5 |
| spawn grace | until the pod leaves its home box, or 2 minutes | 9.1.5 |
| Hit Report | about 1 s after the Shot that caused it | 9.1.2 |
| kill-credit window | a hit counts for 20 s | 9.2.8 |
| purchase menu without a reply | re-opens itself after a second | 8.7 |
| palette cross-fade | about 50 s | 8.20 |
| rain ramp | one step every 15 s | 8.20 |
| guppy leg, home corner to a nearby point | 14–27 s | 9.4.1 |
| no-traffic watchdog | warnings from 30 s without a frame; the game ends at about 50 s | 3.1 |
| frame processing | at most 5 frames per receive call, about 10 calls per second | 3 |

**Server obligations with a time element.**

| obligation | requirement | § |
|---|---|---|
| Start Session | at once after launch; less than 4096 bytes in total sent before the client's first frame | 7.1 |
| Release Pod at session start | in answer to each state-4 report, until the state changes | 7.1 |
| Release Pod after a death | only in answer to the Drop Request | 7.5 |
| Ping Reply | at once, not queued | 8.7 |
| any frame | at least one to each client every 30 s | 3.1 |
| palette changes | at least 50 s apart | 8.20 |
| after Quit | a pause of a few seconds before `05 "A\r"` | 5.5 |
