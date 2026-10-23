# CyberStrike Implementer's Guide

**Version 1.0 — Part III of III.**

Practical advice for building a CyberStrike server from the [Protocol Specification](protocol.md) (Part I). Section references are to Part I unless marked "Part II" (the [Server Policy](server-policy.md)), and each is a link.

## 1. A minimal server

About a third of Part I is enough for a server two players can log into, fight on and leave. Read these in order and skip the rest until you want the rest of the game:

| step | read | you get |
|---|---|---|
| 1 | [§2](protocol.md#2-overview), [§3](protocol.md#3-transport) | the transport, and the two phases on one connection |
| 2 | [§5](protocol.md#5-chamber-phase) | logon, the Chamber terminal, launch, and the return after a game |
| 3 | [§4](protocol.md#4-framing-and-data-types), [Appendix A.1](protocol.md#a1-one-client-session-start) | framing and the three number encodings; A.1's hex is a test for all of them |
| 4 | [§6.1](protocol.md#61-coordinates-and-orientation)–6.3 | coordinates, ids and slots |
| 5 | [§7.1](protocol.md#71-starting-a-session), [§7.2](protocol.md#72-introducing-players), [§7.5](protocol.md#75-death-and-respawn), [§7.6](protocol.md#76-leaving) | starting a session, introducing players, death and leaving |
| 6 | [§8.3](protocol.md#83-a--release-pod-and-drop-request), [§8.4](protocol.md#84-b--start-session), [§8.11](protocol.md#811-i--damage-and-hit-report), [§8.12](protocol.md#812-j--destroy-entity-and-pod-died), [§8.24](protocol.md#824-b--pod-state), [§8.25](protocol.md#825-c--shot), [§8.7](protocol.md#87-e--extended-messages) (Ping) | the core messages |
| 7 | [§9.1](protocol.md#91-combat), [§9.2.9](protocol.md#929-death) | combat, and what to send when a pod dies |
| 8 | [Appendix A.2](protocol.md#a2-two-players), [Appendix B](protocol.md#appendix-b-audience-summary) | a two-player capture, and who receives what |

For the rest of the game, read [§9.2](protocol.md#92-scoring-and-progression)–9.6 in order, then [Part I §10](protocol.md#10-server-obligations) for every value a server must choose, and Part II for one set of answers.

### 1.1 What a minimal server must do

1. Bridge the serial port to TCP with an 8-bit clean link.
2. Run the logon: wait for `HHH`, send `U#=`, read the account line, send `*`.
3. On `M 1380;3`, send `05 "A\r"` (Part II sends `05 "PRESS RETURN\r"` first; `A` alone is untested). On `/PLAY`, send `05 "E 1 1\r"`, wait for `/GO 0`, send `05 "C\r"`.
4. Send Start Session at once, and repeat it until the client's first frame, keeping the total sent before that frame under 4096 bytes.
5. Answer every Pod State with state 4 with Release Pod, until the state changes.
6. Send Bay Status, then Score Status.
7. Answer every Ping at once. The replies also keep the client's 30-second no-traffic watchdog quiet.
8. When a second player's first Pod State arrives, introduce each to the other, each Introduce Pod followed by Release Pod.
9. Relay Pod State (with the state rewritten to 0 or 1), Shot, Grenade, and Chat (as Text, to everyone including the sender). Without the Grenade relay, a grenade hurts only its thrower.
10. Relay each Hit Report naming a pod to that pod's client as Damage.
11. On Pod Died: Destroy Entity to every client, then Set Bounty 0. Answer the later Drop Request with Release Pod to every client, and send Set Bounty 0 again once the pod leaves drop state.
12. On Quit Request: send Quit, read `QUIT\r` twice, wait a few seconds, send `05 "A\r"`.
13. On a disconnect, send Remove Entity to the other clients.

### 1.2 Stubs

Everything else can be stubbed, so that the client is never left waiting:

| client message | minimal answer |
|---|---|
| Tower Order | Tower Order Reply 3 ("no relay towers"); the multiplier point is refunded |
| Module Order | Module Order Reply with `ok` 0 |
| Air Support Request | Air Support Reply 0, and nothing else |
| Effect Requests, Drone Report, Module Operation, Leg Arrived, Pod Flags, Acknowledge, Hit Reports on non-pods | ignore |

Without logistics, NPCs or territory values the game is still playable: players walk, fight, die, respawn and score.

## 2. Common pitfalls

Each of these has cost real implementations time. They are listed roughly by how early they appear.

### Transport and framing

- **A 7-bit link.** The Chamber phase works, but every game frame fails silently. If game messages seem to have no effect, check the link first.
- **Splitting frames on the terminator.** Some client frames have none, and several frames can arrive back to back. A framer that splits on `0a`/`0d` drops them; the first casualty is the client's startup announcement. Frame by `LEN`.
- **Raw string lengths.** A length sent without the `0x10` offset makes the client read far past the end of the message. The symptom can be far from the cause: in one case, a pod dropping late.
- **Treating a `num` as fixed-length.** When a value grows from one byte to two, every later field shifts. A message format that seems to "change after a while" is usually this.
- **Echoing the client's Quit Request.** The echo has no tag, so the client never sees the frame end. Build Quit as a new message.
- **Leaving off the terminator.** The client finds the end of a server frame by its `0d`, not by `LEN`.
- **Going quiet.** After 30 s without a frame the client starts warning of network problems, and after about 50 s it ends the game.

### Chamber

- **Taking the first line as the account.** A bare CR comes first. Skip empty lines.
- **Forgetting the default name.** An unmodified client logs on as `JOINGENIE`, with no password. Refuse it, or accept it as a guest name and give each such connection its own callsign ([Part II §3](server-policy.md#3-identity)).
- **Sending a command without the arming byte.** It is printed as text.
- **Coloring the reply to `/CITY`.** The city buttons lose the second half of their action.
- **Echoing the player's typed lines.** The client has already shown them.

### Session

- **Waiting for the client to say it is ready.** It never does. Send Start Session at once; a CR the client sends after a few seconds is a complaint that no session has started.
- **Waiting for a Drop Request at session start.** The client sends none. Release the pod when it reports state 4.
- **Introducing a pod without releasing it.** The other player appears and never moves.
- **Introducing a pod twice.** The client shows two.
- **Releasing a dead pod early.** After a death the pod reports state 4 during its fly-over. Releasing it then drops it mid-animation. Wait for the Drop Request.
- **Not sending Destroy Entity to the dying client.** It shows its death camera and stops there.
- **Using Destroy Entity when a player leaves.** Their pod explodes and respawns on everyone else's screen. Use Remove Entity.

### Motion

- **Relaying state 4.** The client discards the message, so the pod freezes. State 2 is never needed either. Relay 1 for 1 and 0 for everything else.
- **Relaying turn rates as vertical speed, or the reverse.** The meaning of `v1`/`v2` depends on the state; getting it wrong makes pods spin.
- **Inferring flight from altitude.** Pods stand on roofs at about 212 ft.
- **Clamping off-map positions.** A pod outside the city has a negative offset; clamping makes it jump between two places on other screens.
- **Undoing the 0.9 speed scaling with `round(v / 0.9)`.** The client rounds down, so this delivers 1 less than reported for about half of all values. Use a true ceiling of `v × 0x10000 / 0xe666` ([§8.24](protocol.md#824-b--pod-state)). Relaying unchanged is also valid: pods then run 10% slow between updates, which is probably the original look.
- **Expecting dead reckoning to do the smoothing.** Smooth motion comes from sending each pod's speed, not just its position.

### Combat and scoring

- **Testing damage during spawn grace.** Nothing takes damage until the pod leaves its home box or two minutes pass. Move the pod out first.
- **Watching for Sentinel shots on screen.** They are red-orange and easy to miss. Watch the power bar instead.
- **Creating a Shot with kind 0.** The client crashes within a fraction of a second.
- **Treating a Shot or Hit Report as a trigger pull.** NPCs and automatic guns use the same messages.
- **Gating damage on credit.** NPCs become harmless.
- **Sending Set Bounty 0 before Destroy Entity.** Every payout is 0.
- **Not sending Bay Status.** The multiplier never rises and SCORE never grows.
- **A maximum multiplier above 51.** SCORE can go down.
- **Block classes outside 0–3, palettes above 4, rain above 3.** Corrupt scores or a crash.
- **Text with `0a`, `0d` or `0e`.** The frame ends early.

### Vehicles

- **Redirecting a guppy mid-leg.** The replaced leg is never reported, and leg counting falls apart.
- **Sending a guppy's next destination to every client when the first report of a leg arrives.** A client whose copy is still flying that leg never reports it, falls a leg behind, and strands the guppy once the faster clients leave. Hold it back and bring it into step like a late joiner ([Part I §7.3](protocol.md#73-late-joiners), [§9.4.1](protocol.md#941-the-guppy)).
- **Sending a Muncher's dispatch to a client without its create.** That client gets a stray Muncher heading for the city origin.
- **Sending a Muncher's destination before its create.** It is created at the wrong point.
- **Removing a Muncher with Remove Entity.** Its id does not fit. Use Destroy Entity.

## 3. Testing against the original client

- **Test one hypothesis at a time.** Most remaining questions are of the form "what does the client do when it receives this message?" Send exactly that, and nothing that could explain the result another way.
- **Log both directions, in hex, with timestamps.** Most of what Part I knows came from correlating the two.
- **End spawn grace before testing damage.** Teleport the pod out of its home box with an own-pod Pod State ([§8.24](protocol.md#824-b--pod-state)).
- **Trust the power bar over the screen** for whether something hit.
- **Two clients reveal what one cannot.** Relays, audiences and attribution only show up with a second client watching.
- **Watch the client's Score Status reports.** They are the only view into its scoring, and they arrive every minute or so.

## 4. Anti-cheat starting points

The protocol has no anti-cheat mechanism ([§11](protocol.md#11-trust-and-security-considerations)). If a server does only three things:

1. **Check client-reported progression before saving it.** The client's Score Status and Bay Status are the record of what the player saw ([§9.2.10](protocol.md#9210-reports-and-overwrites)), and progression lasts between sessions and controls which modules a player can use. So check each report against what the server knows before keeping it: SCORE and bay points should rise no faster than the team's territory income and the kill credits the server can account for, and bays should change one level at a time, at the thresholds in [§9.2.7](protocol.md#927-bays-and-progression) or after a Kill Notice or a death. Log what fails rather than acting on it at first.
2. **Check each Shot's `shooter`** against the sender's own pod and the NPCs the server created. It costs nothing.
3. **Log the plausibility checks** in [§11.4](protocol.md#114-what-the-server-can-check) without acting on them. Tune them against honest players first. Destroy Entity is always available later.
