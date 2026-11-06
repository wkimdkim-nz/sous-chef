# sous-chef

A CLI that watches a project management system for development-ready work and drives an AI coding agent through it, from pickup to a merged-ready pull request.

## Language

**Tracker**:
The project management system where work is planned, such as Jira or GitHub Issues.
_Avoid_: PM tool, project management software, board

**Work Item**:
A single unit of planned work in a Tracker, such as a Jira issue or a GitHub issue.
_Avoid_: Ticket, task, story, card

**Ready**:
The condition that makes a Work Item eligible for sous-chef to pick up.
_Avoid_: Dev-ready, groomed, actionable

**Coding Agent**:
The AI that performs the development work on a Work Item, such as Cursor or Claude.
_Avoid_: AI, bot, assistant, model

**Run**:
One attempt by sous-chef to carry a Ready Work Item through a Coding Agent to completion.
_Avoid_: Job, session, execution

**Pass**:
One invocation of a Coding Agent within a Run, from prompt to outcome, such as the first development, a retry after failed checks, a Self-Review, or one feedback round.
_Avoid_: Turn (a model turn inside a Pass), session (a resumable conversation spanning several Passes), attempt, iteration

**Claim**:
Taking a Ready Work Item for a Run by moving it out of Ready and into In Dev, so it isn't picked up twice.
_Avoid_: Lock, reserve, grab, pickup (as a noun)

**In Dev**:
The Tracker state meaning a Run holds the Work Item and its Coding Agent is developing it.
_Avoid_: In progress, WIP, active

**In Review**:
The Tracker state meaning the Run's pull request is open for review and sous-chef is iterating on review feedback and CI failures.
_Avoid_: Code review, pending review

**Awaiting Merge**:
The Tracker state meaning a Run succeeded: its pull request is approved and green, and waits for the developer to merge it.
_Avoid_: Approved, ready to merge, done

**Needs Human**:
The Tracker state meaning a Run stopped without finishing (it failed, hit a limit, or needs information) and waits for the developer to act.
_Avoid_: Blocked, failed, stuck, needs info (each is a reason, not the state)

**Status Comment**:
The single comment sous-chef keeps on a Work Item and edits in place to show where its Run stands.
_Avoid_: Progress comment, workpad, tracking comment

**Recipe**:
The steps and instructions a Run follows to develop a Work Item: sous-chef's built-in default, or a repo's override of it.
_Avoid_: Workflow, playbook, pipeline (GitHub Actions workflows are "CI")

**Trusted User**:
A person or bot whose text sous-chef acts on: someone with write access to the repository, or someone the developer has explicitly listed. Text from anyone else is ignored.
_Avoid_: Maintainer, collaborator, authorized user

**Self-Review**:
A review of a Run's changes by a fresh Coding Agent session, before the PR is marked ready for human review.
_Avoid_: AI review, auto-review, pre-review
