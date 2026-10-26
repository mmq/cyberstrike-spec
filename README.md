# CyberStrike Server Specification

**Version 1.0.** What a game server needs to work with the DOS client of *CyberStrike*, version 1.5
(1994), so that the game can be played again. The game ran on GEnie, and later on America Online and
Simutronics' play.net; none of those servers survives. This specification covers the GEnie DOS client,
so that a new server can be written without them.

| part | document | is |
|---|---|---|
| I | [`protocol.md`](protocol.md) | what the client does and requires: transport, every message, the game rules the client implements. |
| II | [`server-policy.md`](server-policy.md) | what the original client leaves up to the server, with a suggested way to handle each. Not normative. |
| III | [`implementers-guide.md`](implementers-guide.md) | practical advice for building a server, starting with a minimal one. |

Sections are cited as `I §n`, `II §n` or `III §n`: the part, then the section number.

It was used to build [OpenCS](https://github.com/mmq/opencs-server), a complete server for the game.

## Notice

CyberStrike was published by Simutronics and played on GEnie, America Online and play.net. This
specification is an independent work, not affiliated with or endorsed by Simutronics, GEnie, America
Online or their successors. It describes the protocol for interoperability, from observation and analysis
of the original game client and from the game's printed manual, and contains none of their code, art or
text. CyberStrike is a trademark of its owner; other trademarks belong to their owners.

## License

Copyright © 2026 Mark Quinn.

The specification is licensed under the [Creative Commons Attribution 4.0 International
License](https://creativecommons.org/licenses/by/4.0/) ([`LICENSE`](LICENSE)): you may copy, adapt and
republish it, including commercially, if you give credit and say what you changed.

The code examples in it (byte sequences, pseudocode and code blocks) are also dedicated to the public
domain under [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/), so they can be used in an
implementation without attribution.

The license covers the text of these documents. It places no conditions on writing a server, or any
other program, that implements the protocol.
