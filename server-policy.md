# CyberStrike Server Policy

**Version 1.0 — Part II of III.**

## 1. About this document

The [Protocol Specification](protocol.md) (Part I) describes what the CyberStrike client does and requires. Many decisions are left open, because the original server made them and that server no longer exists. [Part I §10](protocol.md#10-server-obligations) lists them.

This document suggests a way to handle each of them. **It is not normative:** a server may do otherwise within the constraints Part I states. Each suggestion comes with its reason and, where it matters, the client constraint it respects. Suggestions that follow the game's manual say so.

All behavior described here has been tested with the original client unless stated otherwise.

## 2. Logon

Answer the logon script ([Part I §5.1](protocol.md#51-logon)) as follows:

- On finding `HHH`, it sends `CONNECT\r\n`, then `U#=`.
- On the account line, it sends `PASSWORD: ` at once and `\r\n*` 0.3 s later, without waiting for `<Not Telling!>`.
- It ignores the `<Not Telling!>` line.

`CONNECT` and `PASSWORD:` are not needed by the client; they make the exchange look like the original service.

## 3. Identity

- **Accounts.** A logon with a password (`NAME,PASSWORD`) is a real account. The first logon with a name registers its password; later logons must match it. A second simultaneous logon with the same account is refused.
- **New accounts.** A new name is 3 to 12 characters of `A`–`Z`, `0`–`9`, `_` and `-`, and not one the server keeps for itself (such as `ADMIN` or `SERVER`) or one a guest could be given (such as `JOINGENIE2`); its password is 6 to 64 characters. Registration can be turned off, leaving the operator to create accounts. The number of accounts is capped, and an account never played is removed after 30 days.
- **Wrong passwords.** After five wrong passwords for one account, from any address, that account is refused for a few minutes. The right password afterwards clears the count.
- **Refusing a logon.** In place of the `*` prompt the server sends one line saying why (for example `WRONG PASSWORD` or `THAT ACCOUNT IS ALREADY LOGGED IN`) and closes the connection. The client displays the line ([Part I §5.1](protocol.md#51-logon)).
  *Reason:* the client offers no sign-up step, so first use registers. A real account is the only identity stable enough to restore a game after a line drop (§13). The limits exist because the server is open to anyone who can reach it: they keep the account table bounded and make guessing a password slow. *The limits are tested with scripted clients; the refusal line with the original client.*
- **Guest accounts** (optional). A configured list of names are guest names. A logon with a guest name is accepted without checking any password (a guest name other than `JOINGENIE` still arrives with one, which is ignored). **Guest names are the only names more than one connection may use at once**: each connection is given its own callsign (below). **Guests have no saved progression** (§11.3). The default configuration lists `JOINGENIE` only; an empty list turns guests off, and a `JOINGENIE` logon is then refused. At most 2 guests from one address, and 24 in all, are online at once.
  *Reason:* an unmodified client logs on as `JOINGENIE`, with no password; accepting it as a guest lets someone try the game before configuring a name. Sharing a name is safe only where nothing depends on the name being one person's, which is also why a guest's progress is not kept: its callsign depends on connection order, so a saved record would pass to whoever next took that callsign.
- **Callsigns** are taken from the logon name, since the client has no step for choosing one ([Part I §5.1](protocol.md#51-logon)). They are unique server-wide, upper case, and at most 12 characters. **A real account's callsign is its name**: a new name is at most 12 characters and never one a guest could be given, and guests pass over every registered name. A guest's callsign is its guest name, cut to 12. If that callsign is already online or registered, the server tries the name's first 11 characters followed by `2`, then `3`, up to `9`, then its first 10 followed by `10` to `99`, then its first 9 followed by `100` to `999`, and uses the first that is free (`JOINGENIE` plays as `JOINGENIE`, then `JOINGENIE2`, `JOINGENIE3`; a guest name `ABCDEFGHIJKLM` as `ABCDEFGHIJKL`, then `ABCDEFGHIJK2`, and `ABCDEFGHIJ10` from the tenth). If all are taken, the logon is refused.
  *Reason:* the client's roster shows 12 characters of a callsign. Players chose their callsigns on the original service; a server that adds a way to choose one, at registration or after logon, may allow mixed case.

## 4. The lobby

### 4.1 Entering and leaving

- When the client sends `M 1380;3` (the GO button), the server sends `05 "PRESS RETURN\r"` and waits up to 5 s for the version reply. It logs the reply and **never refuses** a version, since only one exists.
- It then sends `05 "A\r"` (the Chamber buttons) and a welcome line.
- `/QUIT` sends `05 "B\r"` (the GEnie buttons) and the `*` prompt. `BYE` at that prompt closes the connection.
- A player who types nothing in the Chamber for an hour (15 minutes for a guest) is disconnected.

### 4.2 Commands

| line | reply |
|---|---|
| `/HELP` | the command list, followed by `05 "A\r"` again in case an earlier one was lost |
| `/CITY` | one line per city: letter, name, players/capacity, and anything special running there. **Plain text only, no command lines** ([Part I §5.3](protocol.md#53-player-input)) |
| any line ending in `/CITY` (such as `draft/CITY` or `/PLAY /CITY`) | treated as a bare `/CITY`, since a button has typed onto an unfinished line. The draft is not relayed; the reply starts `NOT SENT: <draft>` |
| a bare city letter, within 2 minutes of a bare `/CITY` | a hint: `TYPE /CITY B … /PLAY B …`. Once the prefix is lost, a request for information and a request to play cannot be told apart |
| `/CITY <letter>` | that city's teams, their sizes and callsigns |
| `/STATUS`, `/WHO` | every connected player and where they are: the Chamber, or a city and team |
| `/WHO <callsign>` | that player's location and saved record: SCORE, bay points, bays, battle marker. A guest has no saved record |
| `/SCORE` | the player's own record (§11): last game's SCORE, career total, bay points, bays, marker. A guest is told that guests' progress is not kept |
| `/PLAY` | join the most populated city with room, preferring the default city (B) on a tie |
| `/PLAY <letter> [team]` | join that city, on the requested team if it has room, otherwise balanced (§6), with a line saying so. Teams may be named `RED`, `BLUE`, `GREEN`, `BLACK`, `R`, `G`, or 0–3. `B` means BLUE, so Black needs its name or `3` |
| `/RESUME` | return to a game interrupted by a line drop (§13) |
| `/COLORS` | the seven usable text colors (a testing aid) |
| any other `/word` | "unknown command" |
| any other line | relayed to every other player in the Chamber as `CALLSIGN: text` |

`/HELP`, `/RESUME`, `/COLORS` and the chat relay are not in the client at all ([Part I §5.3](protocol.md#53-player-input)). The manual documents `/HELP` and `/RESUME`; the chat relay is suggested by the game's README, which mentions operators being present in the Chamber.

### 4.3 Chat relay

- A plain line is sent to every other player in the Chamber, not to players in a game, and not back to the sender, whose client has already displayed it.
- Every byte outside `0x20`–`0x7e` is removed before relaying. A `0x01` or `0x05` would put the receivers' clients into command mode.
- Relayed lines are kept short, well within the 512-byte command-line limit.
- Chat is rate-limited per connection and per address. A receiver whose connection has fallen far behind (a modem reads about 1 KB/s) misses chat lines rather than being disconnected; game messages are never skipped. *Tested with scripted clients.*

### 4.4 Colors

Server text uses color `D2`, headings `D6`, chat `D4`, errors `D3`. Each block ends with `D0`, the client's own default color, so the player's typing stays distinct from server text.

### 4.5 Launch

`/PLAY` sends `05 "E <map> <map>\r"` (the map number repeated as the revision), waits for `/GO <status>`, and on 0 sends `05 "C\r"`. On 1–3 it sends a line naming the problem, and the player stays in the Chamber.

## 5. Cities

A **city** is a joinable game: a letter, a name and a set of rules. It is played on a **map**. Everything kept per game (the running world, player counts, held `/RESUME` places) is keyed by the city's letter, so several cities may share one map.

The three cities follow the manual's training, intermediate and advanced tiers. *Reason:* the manual says beginners enter training cities with limited options, and higher cities add weather, power changes, higher multiplier ceilings and stronger modules. The numbers are not from the manual.

| letter | name | map | max multiplier | tower output | max module level | weather | NPCs |
|---|---|---|---|---|---|---|---|
| A | CYBERCITY ONE | 1 | 8 | normal | 2 | clear | none |
| B | CYBERCITY TWO | 2 | 20 | normal | 4 | rain comes and goes | 2 Sentinels, replaced 300 s after destruction |
| C | CYBERCITY THREE | 3 | 40 | low | 5 | storms; palette 3 used as dusk | 4 Sentinels (180 s); a free Muncher every 600 s |

City B is the default. Every value is configuration; each city may also ban modules (§15).

**Block values.** Each city's 8 × 8 map of block classes is configuration, sent whole to every client on joining. Without one, every block is worth 1. The shipped maps are symmetric across the four corners, low at the edges, where the home corners are, and higher toward the center:

- A: a 4 × 4 center worth 2;
- B: rings worth 1, 2, 5 and 10;
- C: rings worth 2, 5 and 10, with a 4 × 4 center worth 10.

*Reason:* symmetry keeps the four teams equal; a valuable center makes territory something to fight over rather than hold at home. Each grid's total × the city's maximum multiplier (640, 3 360 and 12 640) is below the overflow limit ([Part I §9.2.6](protocol.md#926-territory-income)).

**Generated maps** are supported but off by default: a player can use one only if the map file has been given to them beforehand.

**Weather.** Each city has a weather program: a list of steps, each taken a set time after the one before, repeating from the start when it ends. The city's game starts the program when it begins. Players present get each change as a transition: a palette by cross-fade (Ambient subtype 1), rain by ramp (subtype 3), lightning by subtype 4. A player who arrives later gets the current state at once (subtypes 0, 2 and 4). Palette changes are spaced at least 50 s apart, including across the program's wrap ([Part I §8.20](protocol.md#820-r--ambient)).

| city | program |
|---|---|
| A | none: palette 0, no rain |
| B | after 600 s, rain to 2; after 300 s more, rain to 0; repeat |
| C | after 300 s, palette 3 ("dusk"); after 300 s, rain to 3 with lightning on; after 240 s, rain to 0 with lightning off; after 300 s, palette 0; repeat |

## 6. Teams and capacity

- A requested team is honored if it has room (four players). Otherwise, or with no request, the player joins the smallest open team. The policy is configurable per city: honor-or-balance, always balance, or random (the manual's wording).
- By default all four teams are open. A city can be configured to fewer; a test configuration might open only Red, so that testers land together. Two players who each type `/PLAY <city> RED` also land together.
- Capacity is 16 per city.
- There is no rank gating.

*Reason:* the manual says `/PLAY` picks a random team and a requested team is honored if capacity allows; balancing was preferred to randomness.

## 7. Session timing

| event | reference behavior |
|---|---|
| Start Session | sent every 0.2 s from launch until the client's first frame, capped at 3500 bytes so as not to overrun the client's 4096-byte receive buffer while the engine loads |
| Release Pod at session start | sent with every reported state 4, about 5 per second |
| Release Pod for other pods | each client is sent Release Pod for a pod right after its Introduce Pod, and again after that pod's owner sends a Drop Request; never in answer to the state-4 reports of a pod's fly-over ([Part I §7.5](protocol.md#75-death-and-respawn)) |
| Drop Request after a death | answered at once. If none arrives within 20 s, the server releases the pod anyway |
| Quit | waits up to 10 s for the first `QUIT\r` and 1 s for the second; sends `05 "A\r"` 3 s after the last one seen; closes the connection if none arrives |
| tags | a sequence number per connection, `0x10` to `0xff`, then back to `0x10`. Acknowledge is used to measure round-trip time |
| client Pod Flags | not relayed |
| relayed Pod State speeds | the client's 0.9 scaling is undone: `fwd`, `fwd_target` and, when relaying `state` 0, the turn rates `v1`/`v2` are each sent as ⌈v × `0x10000` / `0xe666`⌉ (a true ceiling), then clamped (`fwd`/`fwd_target` to −120..119, turn rates to −32768..32767), so other pods move at their reported speeds ([Part I §8.24](protocol.md#824-b--pod-state)). *Reason:* on a low-latency link the overshoot after a sudden stop is small. The original server probably relayed unchanged. Not yet compared in play |

## 8. Entity ids

| range | used for |
|---|---|
| 0–239 | players' pods |
| 250–259 | guppies |
| 260–4095 | Munchers |
| 4096–8191 | resident Sentinels |
| 8192–32767 | towers, loose modules, mines, called Sentinels |

Ids are issued in order. When a range runs out, the earliest-released id is reused, but only if every connected client's game engine started after that id was released ([Part I §6.2](protocol.md#62-entity-ids)). *Not yet tested live: no session has run long enough to exhaust a range.*

*Reason:* only pods need ids below 240, so players get all of them: a simulated busy evening in a long-lived city exhausted a smaller player range. Keeping Sentinels out of that range matters because a city's game never ends (§17) and would otherwise exhaust it.

## 9. Combat

### 9.1 Damage

The damage byte from a Hit Report is relayed **unchanged** to the target's client as Damage, **regardless of who is credited** and regardless of team.

*Reason:* the shooter's client resolved the hit and the victim's client applies it; recomputing the number would replace the client's arithmetic with the server's. Damage and credit are separate: a Sentinel's shot must hurt even though it credits nobody, and gating damage on credit would make every NPC harmless. Friendly fire is applied because nothing in the client's damage path checks teams; payouts still exclude teammates, as the manual says.

### 9.2 Trust

The shooter's client is accepted as authoritative for its own hits. This is how the protocol works and almost certainly what the original server did. It is a known opportunity for cheating, not a defect ([Part I §11](protocol.md#11-trust-and-security-considerations)).

Each Shot's `shooter` is checked against the sender's own pod and the NPCs the server created. A mismatch is logged, and the Shot is still relayed: the check detects, it does not filter.

### 9.3 Hit credit

Each connection has a log of the Shots it sent, **including NPC shots, even if they were not relayed**.

```
on a Shot from connection X naming shooter S:
    append (now, S) to X's log

on a Hit Report from connection X naming target T:
    candidates = shots in X's log, not yet used, sent between 0 and 2 s ago
    if none:                            credit nobody
    else if any candidate is X's own pod:
        use the one nearest 1 s ago;    add X's player to T's hit set
    else:
        use the one nearest 1 s ago;    credit nobody
```

- **A player's own shot is preferred over an NPC's**, even when the NPC's is nearer the expected 1 s delay. A player's shot that hits is almost always reported; an NPC's usually is not. On test data this rule credited 15 of 15 hits correctly, where "nearest wins" managed 9 and "most recent wins" 14.
- Using up one shot per report stops a burst of NPC hits from all being credited to one player shot.
- The log is per connection, so a hit can only ever be credited to that connection's player, or to nobody.
- The hit set is a set, so crediting the same player twice in its window has no effect.

The 0–2 s window is not from the client or the manual.

## 10. Durability

| entity | damage absorbed | destroyed with |
|---|---|---|
| tower | 3000 | Destroy Entity at 0 |
| MegaTower | 6000 | Destroy Entity at 0 |
| Sentinel (resident or called) | 3000 | Destroy Entity at 0 |
| Muncher | 3000 | Destroy Entity at 0 |

Each Hit Report on a structure subtracts the base damage of its class ([Part I §8.11](protocol.md#811-i--damage-and-hit-report)), ignoring magnitude, whoever fired.

*Reason:* the client keeps no health for structures. 3000 is a full pod, so a tower takes the same punishment as a pod; the manual fixes only the 2:1 MegaTower ratio. The starting gun does 150 per hit, so a tower takes about 20 hits.

## 11. Scoring and progression

### 11.1 Following the client

The server does not run its own multiplier schedule or bay rules for a real client. It **mirrors** the client: multiplier from 1 at joining, then from each Score Status, and back to 1 at the player's death; bays and battle marker from each Bay Status.

*Reason:* the client's tick is slower than a second, so a server running its own clock drifts ahead (a kill at 31.9 s was paid ×2 by the server while the client showed ×1). Paying with the client's value keeps the server's record matching what the player saw.

### 11.2 Bounty and payouts

- Each living pod's bounty is accrued at 10 per second (1 per second on the wire) and pushed with Set Bounty. It starts from 0, with a Set Bounty 0, when the pod leaves drop state ([Part I §9.2.9](protocol.md#929-death)).
- On a death: every non-teammate in the victim's 20-second hit set is paid the victim's whole bounty × their own multiplier (the manual's rule). Each is sent a Kill Notice, earned if the victim's highest bay ≥ the payee's lowest bay − 1 (the manual's rule). Then Destroy Entity to every client, then Set Bounty 0. Kill Notices go to different clients, so their order among payees does not matter; each precedes the Destroy Entity on its own connection.
- The payout recorded is what the client credited, not the server's own calculation: the change in SCORE between the payee's Score Status reports before and after the kill. That change also includes any territory income earned in between; the record is for display, so this is accepted.

### 11.3 Persistence

- **Saved** from the client's reports: bays, battle marker, SCORE, bay points. Not the multiplier, which the manual restarts each game.
- **Guests are not saved** (§3). Each game a guest plays starts as a new player's would (bays `1 1 1 1 0 0 0 0 0`, marker 0, bay points 0), and nothing it earns is kept after it leaves.
- **SCORE is per game.** A new game starts at 0; only `/RESUME` restores a SCORE. A separate career total of finished games is kept for `/SCORE`. *Reason:* the README says `/resume` restores SCORE alongside the per-game multiplier, and the manual says only bays and points persist.
- **Restored** at session start: Bay Status with the saved bays and marker, then Score Status with the saved bay points, SCORE 0 (or the held SCORE for `/RESUME`), and the starting multiplier (§12).
- **Stale reports.** Any Score Status whose SCORE is below the last saved is refused. After a restore, every report is held until one shows a SCORE and bay points each at least the restored values, or 10 s pass.
- **Unreported spending.** When a Bay Status shows one bay up one level with the marker unchanged, the server subtracts the threshold for that bay's level before the upgrade ([Part I §9.2.7](protocol.md#927-bays-and-progression)) from the saved bay points, and floors the result at 0. The floor is the usual case: the client upgrades the moment its points cross the threshold, so the last saved figure is almost always just below it. Zero is the only value that cannot re-award the upgrade, and the next Score Status corrects it. *Not yet tested live; checked against replayed captures and a synthetic test.*
- **The player's own death** lowers the saved battle marker by 1, mirroring the client.
- The quit sequence's reports are saved like any other.
- **Loss on a line drop:** whatever accrued since the client's last Score Status, up to about a minute.

## 12. Respawn

- The client's own respawn is used, without limit: the server answers the Drop Request at once and never refuses.
- **Multiplier floor** (the manual's rule): at the start of a new game, and on the first Score Status the client sends at least 2 s after a death, a multiplier below the city's maximum module level is raised to it (capped at the city's maximum multiplier). The server sends a Score Status carrying that report's own SCORE and bay points. `/RESUME` is exempt.
- The manual's "returns to a block controlled by their team" is not done: the client always spawns at its home corner, and moving it safely would need knowledge of open ground the server does not have.

*Reason for the 2 s wait:* after a death, the client reports nothing until its next periodic Score Status; pushing earlier would overwrite SCORE with a figure up to a minute old.

## 13. `/RESUME`

The game's README promises that a player who loses the connection may reconnect within 10 minutes, type `/resume`, and return to the game they were playing (if there is room), with score, multiplier and modules normally intact (depending on the most recent automatic status save), but not if the drop happened in a fight they were about to lose. A server can implement that as follows.

1. **Held place.** Created only by a line drop: the connection closing, or 15 s of silence once the game engine has sent anything. A clean quit or a death creates none. Only real accounts get one; guests are refused, since a guest's callsign depends on connection order.
2. **What is held:** identity, city, team, SCORE, multiplier (less any tower spending since the last report), bay points, the confirmed loadout (see below), the time of the drop, and eligibility. Held in memory.
3. **Eligibility.** A drop is **ineligible**, a "fight about to be lost", if, in the 20 s before it, the pod was named in a non-teammate's Hit Report, was sent Damage, or had an enemy shot, grenade or air strike fired within 600 ft of it (the client's own NPC targeting range). Power is **not** used: power changes from jumps and crashes look like hits, and a pod at full power can still be losing.
   - An ineligible drop is treated as a **death at the moment of the drop**: payouts and Kill Notices to the hitters, Destroy Entity to every other client, bounty zeroed, marker −1 saved, modules lost. Then Remove Entity. **No place is held:** a `/RESUME` is refused with a message saying why, and no team slot is reserved.
   - Any other drop is eligible: it gets only Remove Entity, and its place is held.
4. **Time limits.** The player has 10 minutes from the drop to reconnect and enter the Chamber. Once they enter the Chamber, they have 3 minutes to type `/RESUME` and rejoin the game (as described in the manual). During the 10 minutes and the 3 minutes, the team slot is reserved and the city's game keeps running, even if no one else is in it. `/PLAY` gives the place up. At most 2 places are held for one address at a time.
5. **Restore.** The Chamber launch for the held city, Start Session with the held team and a fresh pod id ([Part I §8.10](protocol.md#810-h--remove-entity)), Bay Status (saved bays and marker), Score Status (held SCORE and multiplier, saved bay points, no multiplier floor), and once the pod has landed, one module install per held module, highest level first. Bounty and position are not restored: the pod drops at its home corner with bounty 0.

**The confirmed loadout** is kept for every player as a ledger ([Part I §9.3.4](protocol.md#934-the-installed-loadout)): each install is pending until a Pod State's weight includes it, and dropped if a couple of reports match without it; Module Operation 1 and 4 remove modules; a death clears them. A mismatch with the reported weight is logged.

*Tested with the original client except the attack-based eligibility rule.*

## 14. Logistics

- **One guppy per team**, id 250 + team, waiting at the center of its home corner block.
- **One job at a time.** Orders queue. A tower order arriving when four jobs of any kind (towers, modules, Air Support) are already waiting is refused with Tower Order Reply 2 ("mission queue full", refunded). Module orders and Air Support are refused, as a safety bound, only at 32 waiting jobs.
- Towers are spawned with `carrier` 0 and fall from the ordered point.
- A tower order outside the city is not refused. The tower is delivered, but a tower outside the city claims no block ([Part I §8.15](protocol.md#815-m--spawn-tower-and-tower-order)).
- **Priority** orders go ahead of every waiting non-priority order of that team, behind earlier priority orders. The job in flight is never displaced.
- After each job, the guppy flies home if nothing is waiting. An order arriving while it flies home waits until it is home: the guppy is never redirected mid-leg.
- **No timeout.** A guppy waits for its Leg Arrived however long it takes. With no client connected nothing reports, and the job waits. The operator has a failsafe that sends a stranded guppy home.
- The manual's delivery rules ([Part I §9.4.3](protocol.md#943-delivery-as-the-manual-describes-it)) are applied in full.
- "Within one block" in module effects means within 600 ft in a straight line. Towers are located by the point they were ordered at, since the client never reports where a tower came to rest.
- A server without logistics would answer every tower order with Tower Order Reply 3 ("no relay towers").

## 15. Module stock

Every module is in stock in every city, and every valid order is accepted, except modules a city bans (none by default). A banned or unavailable module is refused with Module Order Reply `ok` 0.

*Reason:* neither the client nor the manual gives stock counts or per-city lists.

## 16. Effects, Munchers and NPCs

### 16.1 Effect modules

| request | reference behavior |
|---|---|
| Repair | the sender's team's towers within 600 ft restored to full |
| Harden | the sender's team's towers within 600 ft have their remaining and maximum capacity doubled, once (it does not stack) |
| Liberator | enemy towers within 600 ft destroyed with Destroy Entity |
| Immunize | the sender's team's towers within 600 ft immune to Munchers for 30 minutes |
| Mine | a mine (Spawn Mine, variant = team) at the sender |
| Mine Sweep | enemy mines within 600 ft destroyed |
| Weather | the city's palette steps to the next value, 0 → 1 → 2 → 3 → 4 → 0, sent to every client and to later joiners |
| Call Sentinel | a Sentinel spawned at the sender |
| Call Muncher | a Muncher created at the sender (§16.2) |
| Air Support | Air Support Reply 0; the team's guppy flies to the point and, on arrival, fires a grenade (a Shot of kind 9 with the guppy as shooter). Not refused in no-drop zones |

### 16.2 Munchers

- A called Muncher goes for the nearest tower, of any team, within 600 ft of the Muncher that is not immune; latches onto it with Muncher Attack on arrival; eats it (Destroy Entity) after 15 s; and repeats from there.
- With nothing left in range, it flies to the nearest city edge and is removed.
- Its durability, like a Sentinel's, is in §10.
- Each city may also send a **free Muncher** at a fixed interval, from the nearest edge to a tower of the team controlling the most blocks, one at a time (the manual's "often target the team controlling most blocks"). *Not yet tested live.*

### 16.3 Resident Sentinels

Each city has a number of resident Sentinels, placed one per block, outside the 3 × 3 blocks around each home corner. A destroyed Sentinel is replaced after a delay set per city (§5).

## 17. Match structure

**A city's game never ends.** Players come and go; the world (towers, territory, weather) lives while anyone is in it or holds a `/RESUME` place. The next game in an empty city starts fresh.

*Reason:* the manual describes persistent cities players drop into and out of, and names no winner, round or reset; the client has no notion of a match ending. Quit, sent unprompted, would end players' games if rounds were ever wanted.

## 18. Loose modules

- A delivered module is spawned 100 ft above the orderer and 20 ft ahead of its facing at the time of the order (the manual's "about 100 ft above and 20 ft ahead"), with velocity (0, 0, 0).
- An ejected module is spawned 15 ft up with a small push (±1 horizontally, 1 upward).
- A picked-up module is destroyed on the other clients with Module Operation subtype 2.

The velocity units are unknown; these values look right.
