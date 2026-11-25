# Hyggja

Nugent's home repo — the federation's **local-model-builder** seat.

Named (2026-11-25) for the Old Norse verb *hyggja*, "to think/consider/intend" — the
same root as Huginn, Odin's raven of Thought, without the name clash with the existing
open-source `huginn/huginn` AI-agent project. The deliberate complement to **Muninn →
Mímir** (Memory; the BMA/Wyrd engram hypergraph): Nugent = Thought integrates with the
hypergraph = Memory.

## What this seat does

Plans and builds custom **local models** for the federation: the zero-marginal-cost
broad-sweep inference tier feeding the Contextus discovery pipeline, cheap cascade
tiers, and domain-specialized/cost-free local inference more broadly. Carries the
beekeeper's Edda-check → fewer-hallucinations theory as its north star.

## Phase

**Crawl.** Heavy model builds (training/fine-tuning) are hardware-gated to Walk. See
`theory/` and `spec/` for the first design surface under review, and
[`inter#124`](https://github.com/JamesPagetButler/inter/issues/124) for the full
reading list and status.

## Layout

- `theory/` — design-rationale documents (the "why"), reviewed before any spec is built on them.
- `spec/` — concrete technical specs (the "what"), reviewed before anything executes.

See the root workspace's `~/Documents/CLAUDE.md` and `~/Documents/Nugent/CLAUDE.md`
for federation-wide conventions and this seat's charter.
