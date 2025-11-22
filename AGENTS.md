## Status: ✅ COMPLETE

We have successfully implemented and tested npm package publishing using GitHub Actions with OpenID Connect (OIDC) authentication and provenance generation.

**All 4 test phases passed.** See `.memory/summary.md` for complete documentation.

## Research Guidelines

- store findings in `.memory/` directory
- all notes in `.memory/` must be in markdown format
- except for `.memory/summary.md`, all notes in `.memory/` must follow the filename convention of `.memory/<type>-<id>-<title>.md`
- where `<type>` is one of: `research`, `phase`, `guide`, `notes`, `implementation`
- Always keep `.memory/summary.md` up to date with current status, prune incorrect or outdated information.
- when finishing a phase, compact relevant successful outcomes from implementation, research and pahse into the summary.md and delete the other files.
- update `AGENTS.md` to indicate which phase is being worked on and by whom.
- use `TODO.md` to track remaining tasks.
- when committing changes, follow conventional commit guidelines.
- Use clear commit messages referencing relevant files for changes.

## Execution Steps

1. always read `.memory/summary.md` first to understand successful outcomes so far.
2. update `AGENTS.md` to indicate which phase is being worked on and by whom.
3. If there are any `[NEEDS-HUMAN]` tasks in `TODO.md`, stop and wait for human intervention.
4. follow the research guidelines above.
5. when you are blocked by actions that require human intervention, create a `TODO.md` file listing the tasks that need to be done by a human. tag it with `[NEEDS-HUMAN]` on the task line.
6. after completing a phase, update `.memory/summary.md` and prune other files as necessary.
7. commit changes with clear messages referencing relevant files.

## Human Interaction

- If you need clarification or additional information, please ask a human for assistance.
- print a large ascii box in chat indicating that human intervention is needed, and list the tasks from `TODO.md` inside the box.
- wait for human to complete the tasks before proceeding.
