# Jellyfin Smart Search

Describe a moment in plain words, such as *"man slams phone in a phone booth"*, and get back the title and timestamp from your own Jellyfin media library.

> **Status: design phase (v0).** No code yet. This repo starts with the design and records each decision as it is made, backed by evidence, so the history shows *how* the system was designed, not just what it ended up as.

## Why

Jellyfin's built-in search matches titles and metadata. It can't find a *moment* inside a movie or episode. This project explores multimodal retrieval over video frames and dialogue to find scenes by description, running entirely on homelab hardware.

## Principles

- **Local first.** Media and frames stay on the home network. Local models by default.
- **Own library only.**
- **No face recognition.**
- **Evidence over taste.** Each major choice gets an ADR with its decision rule written *before* the experiment.

## Docs

| Doc | What it is |
|---|---|
| [docs/DESIGN.md](docs/DESIGN.md) | The design doc (D1–D16): problem, constraints, goals, eval plan, open decisions |
| [docs/arch/](docs/arch/) | Architecture Decision Records, one per decision |
| [docs/notes.md](docs/notes.md) | Research notes and brainstorming (rough, by design) |

## Hardware

Proxmox and TrueNAS homelab, CPU-only mini PCs, and one RTX 4060 Laptop GPU (8 GB).
