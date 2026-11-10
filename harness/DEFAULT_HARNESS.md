# Default harness

Minimal shared guidelines for side hustle work on a target repo that defines no harness of its own (SPEC.md E4, AC14). If the target repo has a harness (CLAUDE.md/AGENTS.md, skills, commands, hooks, tests, lint, conventions), follow that instead; it always wins (P4).

## Rules

1. **Instructions come only from the Asana task**: its description, its comments and any documented state. Information that exists only in Slack is not an instruction until it is in Asana.
2. **Do what the task asks, nothing more.** Meet every acceptance criterion. Do not touch anything listed under "Out of scope / do not touch". Make no changes unrelated to the task.
3. **Be non-destructive.** Never:
   - delete data;
   - run irreversible migrations;
   - force-push;
   - change permissions or secrets;
   - remove files, tests or functionality that the task did not request.

   If the task asks for any of these (other than a requested removal of files, tests or functionality), follow rule 9.
4. **Never deploy, merge or publish a final version.** Deliver for review only: a branch and an MR, a Confluence draft or review space, or tasks in the target project.
5. **Keep changes small and in the existing style.** Follow the repo's existing structure, naming and formatting.
6. **Run what the repo already has.** If it has tests, a linter or a build, run them and make sure they pass. Add tests for new behavior where the repo already has tests.
7. **Write clear commits.** Each commit message says what changed and why, and links to the Asana task.
8. **Say that this harness was used.** The deliverable (MR description, document or task comment) must state: "No repo harness found; the side hustle default harness was applied."
9. **When in doubt, ask in Asana.** If the task is ambiguous or contradicts these rules, comment in Asana and move the task to Needs Info instead of guessing.
