# Contributing to agicore-examples

Thank you for your interest in contributing. This repository demonstrates agentic patterns implemented with the Agicore DSL — one folder, one `.agi` file, one README per example.

---

## Proposing a new example

Open a GitHub issue with:

1. **Pattern name** — a short, descriptive name (e.g. "Autonomous Code Reviewer")
2. **Problem statement** — what agentic anti-pattern does this replace?
3. **Agicore declarations** — which DSL declarations does it showcase? (WORKFLOW, ROUTER, REASONER, etc.)
4. **Sketch** — a rough outline of the key entities, actions, and flow

Not every proposed example will be accepted. Priority goes to examples that:
- Showcase a declaration type not yet well-represented in the repository
- Replace a genuinely common fragile-agent pattern
- Are self-contained (no external services beyond LLM API keys)

---

## Submitting an example

Each example lives in a numbered folder following the existing convention:

```
NN-example-slug/
  example_name.agi     # the compiled .agi file
  README.md            # explanation, anti-pattern replaced, how to run
```

### Requirements for the .agi file

- Must parse cleanly with the Agicore parser (`node core/parser/dist/index.js parse your_file.agi`)
- Must include at minimum: `APP`, one `ENTITY`, one `VIEW`
- Must use only documented DSL declarations — no invented syntax
- Field names must be `snake_case`; action names must be `verb_noun`
- Triple-quoted strings (`"""..."""`) for AI prompts containing special characters

### Requirements for the README

Each `README.md` must include:

1. **What problem this solves** — one paragraph
2. **The agentic anti-pattern it replaces** — what people do with LangChain/AutoGen today
3. **Key Agicore declarations used and why** — a brief explanation of each
4. **How to compile and run** — the exact commands

### Pull request checklist

- [ ] Folder follows `NN-slug/` naming (use the next available number)
- [ ] `.agi` file parses without errors
- [ ] README covers all four sections above
- [ ] No invented DSL syntax — only declarations documented in the Agicore framework
- [ ] No hardcoded API keys, secrets, or personal data in the `.agi` file

---

## Questions?

Open an issue or start a discussion. If you are unsure whether a proposed pattern fits, ask before investing time in implementation.
