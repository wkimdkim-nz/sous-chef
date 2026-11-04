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
