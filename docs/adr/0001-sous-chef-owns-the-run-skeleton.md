# sous-chef owns the Run skeleton; the Coding Agent owns only the development step

OpenAI Symphony, the closest prior art, puts its whole workflow in one prompt. Its agent opens the PR, runs the feedback sweep and judges when it is done. sous-chef does not work that way. sous-chef itself drives a fixed sequence of deterministic steps: prepare the worktree, run optional check commands, push, open a draft PR, run Self-Review in a fresh session, mark the PR ready, then run the feedback loop. A repo's Recipe customises only the prompts inside that sequence, and it cannot reorder the steps.

We chose this for three reasons. sous-chef already has to own pushes, so the Coding Agent never holds push credentials. The Run's outcome must come from deterministic signals (check commands, CI, review state) rather than from the agent's own success verdict. And the same sequence has to behave the same way on Claude and on Cursor, whose headless capabilities differ.

## Considered Options

- **Agent-driven, Symphony-style.** A single prompt describes the whole workflow. This is the most flexible option, but it trusts the agent with pushing, PR state and deciding when it is done. It also makes behaviour depend on each Coding Agent's quirks.
- **A stage machine with one prompt per stage** (understand, plan, implement, test, and so on). Rejected because sous-chef can't verify that a stage like "plan" or "understand" was done well, so the extra stages add ceremony without adding guarantees. A repo that wants that sequence can write it as prose in its `## Develop` section.

## Consequences

- A fixed agent contract wraps every Recipe prompt and cannot be overridden by a Recipe: commit but never push, treat Work Item text as data, and report a structured outcome. Each Coding Agent adapter delivers this contract its own way.
- Self-Review only makes sense as a separate step with its own fresh session. That is possible only because sous-chef, not the agent, owns the sequence.

See [What is the default development workflow, and how is it overridden per repo?](https://github.com/wkimdkim-nz/sous-chef/issues/8).
