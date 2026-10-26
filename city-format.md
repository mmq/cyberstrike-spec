# CyberStrike Map File Format

**Version 1.0 — Part IV of IV.** Part I is the [*Protocol Specification*](protocol.md); Part II is the [*Server Policy*](server-policy.md); Part III is the [*Implementer's Guide*](implementers-guide.md).

> CyberStrike was published by Simutronics and played on GEnie, America Online and play.net. This specification is an independent work, not affiliated with or endorsed by Simutronics, GEnie, America Online or their successors. It describes the protocol for interoperability, from observation and analysis of the original game client and from the game's printed manual, and contains none of their code, art or text. CyberStrike is a trademark of its owner; other trademarks belong to their owners.

---

## 1. About this document

### 1.1 Scope

This document specifies the format of the client's map files, `C<n>.CBD`, and what the client does with their contents: drawing the city, colliding with it, and launching pods off launch pads. It is written for people who want to make new maps, or to read the shipped ones.

**A server does not read map files and need not implement anything here.** All a server does with a map is name it at launch (I §5.4). Section 6 lists the few points where a map's contents matter to a server anyway.

The map file holds geometry only. Teams, home corners, block values, towers, NPCs, weather and every other rule of a city come from the client's own code or from the server (I §6).

### 1.2 Conformance language

MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as in I §1.3, but apply to **map files** and the tools that write them:

- **MUST / MUST NOT**: the client will refuse the file, crash, or misbehave visibly otherwise.
- **SHOULD / SHOULD NOT**: the map works otherwise, but something is lost, or the result is untested.
- **MAY**: an option that works with the client.

Some rules are marked **(precautionary)**. For these, what the client does when the rule is broken is not known, but obeying the rule costs a map maker nothing, so it is stated as a MUST anyway.

### 1.3 Confidence grades

Rules carry the grades of I §1.4: **confirmed**, **established**, **inferred** and **unknown**. **Server-defined** does not apply here. One grade is added:

| grade | meaning | relying on it |
|---|---|---|
| **convention** | Every shipped map does this, and maps that follow it work. The client is not known to require it. | Follow it unless you are testing it. |

"Confirmed" here means a real client has loaded and played a map built from this specification, from scratch, and behaved as described.

### 1.4 Notation

- All multi-byte values are **little-endian**.
- `u8`, `u16` are unsigned integers of 1 and 2 bytes. `fix` is a **signed 32-bit 16.16 fixed-point** number: the value times 65536, so 1.0 is `00 00 01 00`.
- Coordinates are in **feet**, the unit of I §6.1, with the same axes: X and Z horizontal, Y up. The city spans 0 to 4800 ft in X and Z.
- A **block** is one 600 ft × 600 ft square of the 8 × 8 city grid (I §1.5), named by (block X, block Z).

---

## 2. Overview

A map is the city's 64 blocks, each described on its own. Each block carries three independent lists:

- **vertices**: points in world coordinates;
- **faces**: flat polygons built from those vertices, each with a material (color) and a normal. Faces are what the player sees;
- **collision volumes**: boxes and ramps. Volumes are what pods run into and stand on.

**Faces and volumes are not linked.** Nothing in the file connects a face to a volume, and the client never relates them. A building is visible because of its faces and solid because of its volumes; a map that gives it only one of the two gets an invisible wall or a building pods walk through. *Established.*

**The ground and the sky are not in the file.** The client draws both itself, and a pod walks on the ground at altitude 0 wherever no volume is in the way. *Confirmed.*

```
file
├── header            14 bytes
└── block × 64        variable length, in a fixed order
    ├── vertices
    ├── faces
    └── volumes
```

The shipped maps, for scale:

| file | bytes | vertices | faces | volumes |
|---|---|---|---|---|
| `C1.CBD` | 33 780 | 989 | 575 | 147 |
| `C2.CBD` | 30 633 | 962 | 504 | 121 |
| `C3.CBD` | 33 838 | 997 | 569 | 153 |

---

## 3. File layout

*Established unless marked.* Every shipped map decodes with this layout with no bytes left over.

### 3.1 Header

| offset | size | field | value |
|---|---|---|---|
| 0 | 6 | magic | the text `CSCBD2` |
| 6 | 2 | revision | two ASCII digits |
| 8 | 4 | padding | the text `----` |
| 12 | 2 | `checksum` | `u16`, see §3.6 |

- **The magic MUST be `CSCBD2`.** Otherwise the game engine stops with the fatal error `Invalid city data`. *Established.*
- **The first revision digit MUST be `0`.** At launch the Chamber program compares bytes 0–6 of the file with `CSCBD2` and the first digit of the server's two-digit `<rev>`, and reports `/GO 2` if they differ (I §5.4). The server sends a `<rev>` from 0 to 9, so the digit is always `0`. *Established.*
- **The second revision digit is not checked by anything.** The shipped maps put the last digit of the map number there (`C1.CBD` has `01`); maps numbered 4 and 5, with `04` and `05`, loaded. *Convention.*
- The padding is never checked. It SHOULD be `----`. *Convention.*

Because only the first digit is compared, **the header cannot tell two versions of the same map apart.** See §6.

### 3.2 Blocks

The header is followed by exactly 64 block records, one per block, in this order:

```
for blockX in 0..7:
    for blockZ in 0..7:
        block record
```

So the second record is block (0, 1), and the ninth is block (1, 0). Records vary in length, so a reader must parse each one to find the next. The game engine reads the file in a single pass, and anything after the 64th record is ignored. *Established.*

### 3.3 Block record

```
vert1:u8  vert2:u8  vertex × (vert1 + vert2)
faces:u8  face × faces
vols:u8   volume × vols
```

| field | meaning |
|---|---|
| `vert1`, `vert2` | the sizes of two groups of vertices, stored one after the other. The client keeps the two counts separately, so the split means something, but what is **unknown**. Every shipped map has `vert2` = 0 |
| `faces` | number of faces in the block |
| `vols` | number of collision volumes in the block |

- **`vert2` MUST be 0, with every vertex in the first group.** *(Precautionary.)*
- **A block MUST have at least one vertex.** The client allocates the vertex list with its exact size, and an empty block fails with the fatal error `OUT OF MEMORY : PointList::SetSize`, however much memory is free. *Confirmed.*
- **A block SHOULD have at least one face and one volume.** Every shipped and tested block has; a block with none is untested. A map that wants an empty block can use a small post. *Unknown.*
- Each count is one byte, so a block holds at most **255 vertices, 255 faces and 255 volumes**. Shipped blocks use at most 48, 30 and 10. *Established.*

### 3.4 Vertex

```
x:fix  y:fix  z:fix                       12 bytes
```

A point in **world** coordinates, not relative to the block. *Established.*

- **Every vertex SHOULD lie inside its block**: X from 600 × block X to 600 × block X + 600, and likewise for Z. A vertex may lie on the block's edge. *Convention*: no shipped vertex is outside its block.
- Shipped maps use whole feet only, and altitudes from 0 to 300 ft. The format allows fractions. *Convention.*

### 3.5 Face

```
flags:u8  material:u8  count:u8  index:u8 × count  ref:fix×3  normal:fix×3       3 + count + 24 bytes
```

| field | meaning |
|---|---|
| `flags` | the client keeps bit 0 and ignores the other bits. Its meaning is **unknown**. Every shipped face has 1 |
| `material` | the face's color, §4.3 |
| `count` | number of vertices in the polygon. Shipped faces have 3 to 9, nearly all 4 |
| `index` | the polygon's vertices, as positions (from 0) in **this block's** vertex list |
| `ref` | `normal` added to the first vertex, `vertex[index[0]] + normal` |
| `normal` | the face's outward unit normal |

- **`flags` MUST be 1.** *(Precautionary.)*
- **Each `index` MUST be less than the block's vertex count.** *Established.*
- **`ref` MUST equal `vertex[index[0]] + normal`**, component by component. It holds for every shipped face. *(Precautionary.)*
- **`normal` MUST be the face's outward unit normal**, its components 16.16 values whose squares sum to 1.0. The client uses it to shade the face (§4.3), so a wrong normal gives a wrong brightness. *Established.*
- **The vertices MUST run so that the right-hand rule gives the inward direction.** Equivalently, for a convex face, the cross product `(v1 − v0) × (v2 − v1)` points opposite to `normal`. With Y up, an upward-facing roof lists its corners clockwise when seen from above. Every shipped face follows this rule. Whether the client depends on it is **unknown**. *(Precautionary.)*
- **A face SHOULD be flat and convex.** All but three of the 1 648 shipped faces are. One is bent by 3 ft, and two are concave; whether the client draws those correctly is **unknown**. *Inferred.*
- **A face's vertices MUST lie in its block (§3.4).** A face that crosses a block edge is split into one face per block. *Convention*: no shipped face crosses an edge. The client stores faces by block, and a face outside its block may not be drawn when it should be. *Inferred.*

### 3.6 Collision volume

```
min:fix×3  size:fix×3  shape:u16                    26 bytes
```

| field | meaning |
|---|---|
| `min` | the corner with the smallest X, Y and Z |
| `size` | the volume's extent along X, Y and Z. **Not** the far corner: the far corner is `min + size` |
| `shape` | the shape, and whether the volume is a launch pad. §5 |

- **Each `size` component MUST be positive.** *Established.*
- **A volume SHOULD lie within its block.** A volume that crosses a block edge is split into one volume per block. *Convention*: two shipped volumes cross an edge by 20 ft, and no others do. Whether the client tests a pod only against the volumes of its own block is **unknown**; if it does, the part of an unsplit volume outside its block has no effect.

### 3.7 The checksum

The header's `checksum` is computed over **every byte of the 64 block records**, from offset 14 to the end of the last record:

```
sum = 0                                        16-bit, wrapping
for each byte b:
    sum = sum + sign_extend(b)                 b as a signed byte, −128..127
    sum = rotate_left(sum, 1)                  16-bit rotate
    if b has an odd number of 1 bits:
        sum = sum + 1
```

- **`checksum` MUST equal this sum.** The client computes it as it reads the file and records a mismatch. What it does about a mismatch is **unknown**. *(Precautionary.)* The sum itself is *established*: it reproduces the checksum of every shipped map.
- Treating `b` as unsigned gives different results. The sign extension matters.

The shipped maps' checksums, for testing: `C1.CBD` 1687 (`0x0697`), `C2.CBD` 17157 (`0x4305`), `C3.CBD` 3303 (`0x0ce7`).

A step-by-step example is in Appendix A.

---

## 4. Drawing

*Established unless marked.*

### 4.1 What is drawn

The client draws each block's faces. Faces are polygons with one flat color each; there are no textures. The ground and sky are drawn by the client, not the map (§2).

Every shipped map models buildings as closed solids with outward faces only. Whether the client draws a face seen from behind is **unknown**. A face meant to be seen from both sides SHOULD be given twice, once with each normal. *Inferred.*

### 4.2 Shading

Each face is shaded once, from the angle between its normal and a fixed light direction set by the client: the more directly a face is lit, the brighter its color. The shade is one of eight steps. The light direction is not in the map and is not known. *Established*, except the direction.

### 4.3 Materials

The game's 256-color palette is sixteen **ramps** of sixteen colors each. A face's `material` *m* selects ramp *m*, and the face is drawn in one of that ramp's last eight colors, palette entries 16*m* + 8 to 16*m* + 15, chosen by its shade. The more brightly a face is lit, the lower the entry. *Established.*

The shipped maps use eight materials:

| `material` | color in normal daylight |
|---|---|
| 1 | gray-blue |
| 2 | red |
| 3 | green |
| 4 | olive yellow |
| 5 | blue |
| 7 | orange-brown |
| 8 | magenta |
| 9 | near-black gray |

- **`material` SHOULD be one of these eight.** Other values select other parts of the palette, some of which the client uses for other things: ramp 0 holds the standard EGA colors used by the interface, and ramp 12 the ground and sky. They are untested as materials. *Inferred.*
- The colors are the daylight palette's. The server can change the weather with Ambient (I §8.20), which switches the whole palette and changes every material's color; the player's InfraRed module turns every material the same dark green. *Confirmed* for the daylight palette.

The palette is not in the map file. It is resource 200 of the client's `R1.CRS` (daylight), with resources 201, 202, 204 and 205 for the other weather conditions, 203 for InfraRed and 206 for the interface. Each is a 768-byte table of 8-bit red, green and blue values at offset 49 of the resource, which the client divides by 4 for the VGA. *Established.*

---

## 5. Collision volumes

*Established unless marked.*

### 5.1 What collides

A pod cannot enter a volume. Pods stop at walls, stand on roofs, walk up ramps, and walk along raised walkways. A pod standing on a surface reports an altitude about 12 ft above it (seen once). Falling power towers land on roofs (I §8.15). *Confirmed*, on maps built from this specification.

Whether shots, guppies or other entities use the volumes is **unknown**.

### 5.2 Shapes

`shape` is a `u16`:

| `shape` | volume |
|---|---|
| 0 | a solid box |
| 1–4 | a ramp: a wedge filling the box under one sloping face (§5.3) |
| 5–100 | **no shape.** The client never treats a point as inside such a volume, so it has no effect |
| 101 | a solid box that is a **launch pad** (§5.4) |
| 102 and above | a solid box. The client sets its pad number to `shape − 100`, which has no effect |

- **`shape` MUST be 0–4, or 101 for a launch pad.** Values from 5 to 100 make a volume that does nothing.
- On values 0–4 the client never sets the pad number, and uses whatever value happens to be in memory. A box or ramp that happened to receive 1 would act as a launch pad. This has never been seen, and the shipped maps use 0–4 throughout. A map that must rule it out for a box can use `shape` 102. *Established*; the effect is unobserved.

### 5.3 Ramps

A ramp fills its box below a plane that rises from the bottom of the box at one edge to the top at the opposite edge:

| `shape` | high edge | low edge |
|---|---|---|
| 1 | Z = `min.z` | Z = `min.z + size.z` |
| 2 | X = `min.x + size.x` | X = `min.x` |
| 3 | Z = `min.z + size.z` | Z = `min.z` |
| 4 | X = `min.x` | X = `min.x + size.x` |

So shape 3 rises toward +Z, shape 1 toward −Z, shape 2 toward +X and shape 4 toward −X. A point is inside the ramp if it is inside the box and not above the plane.

The shipped maps use ramps in two ways: four of them (one of each shape) around a box to make a tapered plinth, and single shallow ramps covering a whole block. Ramp faces are drawn by ordinary faces, which the map must supply separately (§2). *Convention.*

### 5.4 Launch pads

A pod touching a launch pad (`shape` 101) is thrown into the air: on a map built from this specification, a pod crossed about 1 000 ft in about 5 s, reaching an altitude of about 177 ft. *Confirmed.*

- **A pad acts only on the player's own pod.** Each client launches its own pod, and the other players see it fly through its Pod State reports (I §8.24). Nothing is sent to the server. *Established.*
- The launch is weaker by an amount taken from a value in the client that may be the pod's weight; the manual says weight reduces jumping. *Inferred.*
- Every shipped map has exactly four pads, each 80 × 1 × 80 ft and lying flat. The client does not require any number of pads. *Convention.*
- **Pads are not spawn points.** A team's pods spawn in its home corner block whatever the map holds (I §6.3), and only `C3`'s pads are in those blocks. *Established.*

---

## 6. Maps and the server

A server never reads a map file, but a map's contents matter to it in four places.

- **Every player in a city MUST have the same map file.** The map is loaded from each player's own disk, and positions are relayed between clients unchanged (I §6.1). A player with a different file sees the other pods inside walls or in the air. The server cannot send a map, and cannot check the player's copy: the header's revision check passes for any copy (§3.1). A server operator distributing a new map SHOULD give it a new number rather than change an existing one. *Inferred.*
- **Home corner blocks SHOULD be kept open.** A team's pods are dropped into their home corner block (I §6.3) at a point that differs from game to game, and land on whatever is there, roofs included. *Confirmed.* A map SHOULD NOT put anything in blocks (0, 0), (7, 0), (0, 7) and (7, 7) that a pod should not land on or in.
- **Geometry above 300 ft SHOULD be avoided where towers can be placed.** A power tower ordered or spawned below 300 ft starts falling from 300 ft (I §8.15), so a roof above 300 ft over the tower's point would be above where the tower starts: the tower would start inside or under the building, and what happens then is unknown. The shipped maps stay at or below 300 ft. *Inferred.*
- **A pod standing on a roof is not flying** (I §6.1). A server that judges a pod's state from its altitude will misjudge it on any map.

---

## 7. Open questions

These are **unknown**. None of them stands in the way of making a map that works.

1. What the client does when the checksum does not match (§3.7).
2. What the second group of vertices (`vert2`) is for (§3.3).
3. What bit 0 of a face's `flags` means (§3.5).
4. Whether the client depends on vertex order, or draws a face seen from behind (§3.5, §4.1).
5. Whether the client tests a pod against the volumes of its own block only (§3.6).
6. How the client draws a concave face (§3.5).
7. Whether a block may have no faces or no volumes (§3.3).
8. What decides where in the home corner block a pod lands (§6).
9. The client's light direction (§4.2).
10. How large a map the client can hold. Maps up to the size of the shipped ones have been tested.

---

## Appendix A. Worked examples

### A.1 One vertex

The point (100, 40, 100):

```
00 00 64 00   00 00 28 00   00 00 64 00
x = 100.0     y = 40.0      z = 100.0
```

### A.2 A roof

A flat roof at 40 ft over X 100–200 and Z 100–200, facing up, material 1 (gray-blue), using vertices 0–3 of its block: (100, 40, 100), (200, 40, 100), (200, 40, 200), (100, 40, 200).

```
01                              flags
01                              material 1
04                              4 vertices
00 01 02 03                     indices
00 00 64 00 00 00 29 00 00 00 64 00     ref    = (100, 41, 100) = vertex 0 + normal
00 00 00 00 00 00 01 00 00 00 00 00     normal = (0, 1, 0)
```

Check the vertex order (§3.5): `(v1 − v0) × (v2 − v1)` = (100, 0, 0) × (0, 0, 100) = (0, −10000, 0), which points down, opposite to the normal.

### A.3 The building under it

A solid box from the ground to the roof:

```
00 00 64 00 00 00 00 00 00 00 64 00     min  = (100, 0, 100)
00 00 64 00 00 00 28 00 00 00 64 00     size = (100, 40, 100)
00 00                                   shape 0, solid box
```

The same box as a launch pad would end `65 00` (shape 101). The box's walls would also need faces to be visible; only its roof is drawn above.

### A.4 The checksum

The checksum over the seven bytes `08 00 00 00 64 00 ff`:

| byte | sum before | after adding | after rotating | odd bits? | sum after |
|---|---|---|---|---|---|
| `08` | `0000` | `0008` | `0010` | yes | `0011` |
| `00` | `0011` | `0011` | `0022` | no | `0022` |
| `00` | `0022` | `0022` | `0044` | no | `0044` |
| `00` | `0044` | `0044` | `0088` | no | `0088` |
| `64` | `0088` | `00ec` | `01d8` | yes | `01d9` |
| `00` | `01d9` | `01d9` | `03b2` | no | `03b2` |
| `ff` | `03b2` | `03b1` | `0762` | no | `0762` |

The last row shows the sign extension: `ff` adds −1, not 255.
