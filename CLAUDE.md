# CLAUDE — xread (`@xresearch/xread`)

Part of the **xresearch** modular monolith. This package's own spec + docs live **here, in this repo** — **Specs = Docs**.

## Role

reader + citation-browser

## Rules (non-negotiable)

- **Pure logic, environment-agnostic.** Never import Cloudflare / Workers / Pages specifics. The **entire** Cloudflare/edge layer lives in `xresearch-app`, never in a package. A non-TypeScript package (Python where the ecosystem demands it — numerics/ML) runs as a **service the app orchestrates** (the GROBID precedent), never as an edge worker. There are no Python workers.
- **Identity = `@xresearch/xread`** — the workspace package name, stable regardless of where this repo physically lives. Imports never change when the repo moves.
- **Compose over the base.** An engine composes only over `xsubstrate` / `xcontract`; a surface may also compose over a peer's *published* `@xresearch/<name>` entry point — never a peer's internals (C-18). That is what keeps every package independently extractable.
- **Conforms to `xcontract`** — the normative request/response shapes; CI diffs the implementation against it.

## UI — every repo has one

- This repo ships **its own UI**: a dev / inspector / demo surface built on **`xkit`** (the shared render component + design system). You see this module's behaviour visually, in isolation. The production Cloudflare Pages deployment is `xresearch-app`'s.
- **For ALL UI development, use the `superpowers` `brainstorming/visual-companion` skill** (`/plugin install superpowers@claude-plugins-official`). No UI work starts without it. Standing convention, no exceptions.