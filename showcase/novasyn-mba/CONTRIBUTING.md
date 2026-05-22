# Contributing to NovaSyn MBA

Thank you for your interest in contributing. NovaSyn MBA serves two purposes: it is a real portfolio OS for solo founders, and it is a reference application for the Agicore framework. Both purposes benefit from community involvement.

---

## Filing Feature Requests

NovaSyn MBA features fall into two categories. Please identify which before filing.

**App-level features** are things the app should do that are achievable with the current Agicore compiler. File these as GitHub Issues with the label `app-feature`. Include the business case: what problem does the solo founder have that this feature addresses?

**Framework-level features** are things that require new Agicore DSL primitives or compiler capabilities (ROUTER, REASONER, SKILL, RULE/FACT, EVENT/TRIGGER, kanban VIEW layout, etc.). File these as GitHub Issues with the label `agicore-gap`. The `.agi` file contains `## TODO(agicore)` comments marking every known gap — these are a good reference for what is already tracked.

Framework gap reports filed here feed directly into the Agicore compiler roadmap. This is intentional: NovaSyn MBA is the idea factory for Agicore features.

---

## Submitting Pull Requests

1. Fork the Agicore repository and create a branch from `main`.
2. Make your changes. If you are modifying `novasyn_mba.agi`, run the Agicore compiler to verify it produces valid output before submitting.
3. If your change touches the DSL file, include the compiler output diff in your PR description so reviewers can see what changes at the generated-code layer.
4. Keep PRs focused. One concern per PR. If you find a bug while implementing a feature, file a separate PR for the bug.
5. Update `DESIGN.md` if your change affects the entity model, AI advisors, or view structure.

---

## App-Level vs Framework-Level Code

The `.agi` file is the source of truth. Do not hand-edit generated files (migrations, Rust commands, TypeScript types, Zustand store slices, generated React components). If the generated code is wrong, the fix belongs in the compiler or in the `.agi` DSL.

Custom React components that extend or wrap generated views live in `src/components/custom/`. These are not overwritten by the compiler.

---

## Code Style

- Rust: `cargo fmt` and `cargo clippy` must pass clean.
- TypeScript/React: Prettier with the project's `.prettierrc`. Run `npm run format` before committing.
- `.agi` files: 2-space indentation, alphabetical field ordering within each entity block, one blank line between top-level declarations.

---

## Code of Conduct

Be direct, be kind, assume good intent. This is a small open-source project maintained by a solo founder. Treat it accordingly.
