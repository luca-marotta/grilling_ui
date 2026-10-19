# Grilling UI

A small web chat on the internal network where each visitor is "grilled" about a project by a headless Claude Code session, one question at a time, until a design template can be filled in. The result is downloadable as Markdown.

Status: **spec only**. No code yet.

- [`docs/SPEC.md`](docs/SPEC.md): the agreed specification
- [`docs/HANDOFF.md`](docs/HANDOFF.md): implementation notes for whoever (or whatever agent) builds it
- `docs/spec.html`: a rendered copy of the spec

## Credits

The grilling behaviour at the heart of this project comes from **Matt Pocock's `grilling` skill**, part of his [skills collection](https://github.com/mattpocock/skills) (MIT License, © 2026 Matt Pocock). This project vendors an adapted copy of that skill into `grill/.claude/skills/grilling/SKILL.md`, and the spec itself was produced in a grilling session run with his skill.

The adapted skill keeps his copyright and MIT licence notice. All credit for the grilling method — the design tree, the frontier, the `➡️` recommended answers — goes to him.
