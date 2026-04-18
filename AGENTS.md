# Autumn Garage — Agent Instructions

Any agent working in this repo should:

1. Read `@.cortex/protocol.md` and `@.cortex/state.md` first.
2. Treat this repo as coordination-only. No application code belongs here.
3. Respect the Append-only Journal and Immutable-with-Supersede Doctrine invariants.
4. Prefer editing existing `.cortex/` files to creating new ones when scope permits.
5. Validate changes with `cortex doctor` before committing.

When in doubt about whether a decision belongs here or in one of the tool repos:

- If it affects **one tool**, it belongs in that tool's `.cortex/`.
- If it affects **two or more tools** or the integration contract between them, it belongs here.

The sibling dogfood project `autumn-mail` has its own `.cortex/` for project-local decisions.
