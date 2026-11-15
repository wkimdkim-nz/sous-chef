# The Tracker and GitHub are the record of a Run, local state holds only what they can't, and shutdown is crash-only

A Run can last for days, and the sous-chef process will stop many times along the way: a crash, a reboot, a closed terminal, an upgrade. Each restart has to carry on every in-flight Run without losing it, without doubling its work, and without resetting its limits. sous-chef doesn't keep a database of record for this. Instead:

- **The Tracker** says whether a Run is in flight, and which phase it is in (In Dev or In Review).
- **GitHub** says everything about its PR.
- **The branch, worktree and PR** are all derived from the Work Item key, so they never need to be stored.
- **A small local file per Work Item** holds only what neither the Tracker nor GitHub can answer:
  - the agent session id;
  - Feedback Rounds and usage against each limit;
  - the last-push time;
  - the current step, and the agent's output that hasn't been posted yet;
  - the process group of the running agent;
  - interruption counts.

When the local file and the Tracker or GitHub disagree, the Tracker or GitHub wins. The local file is never used to decide whether a Run exists.

Recovery is not a separate mode. The regular poll adopts any In Dev or In Review Work Item that nothing is driving. Shutdown is crash-only: Ctrl-C kills the agents, notes "stopped by shutdown", and exits. The next start resumes those Runs exactly as it would after a real crash, so every ordinary restart exercises the crash path.

## Considered Options

- **Keep everything on the Tracker and GitHub**, with counters and the session id in a hidden block in the Status Comment. Rejected. Jira's ADF has no real hidden content, and sous-chef would rewrite the comment after every Pass just to save numbers.
- **A local store as the system of record**, including which Runs exist. Rejected. It can disagree with what the Tracker shows, and the Tracker is what the developer looks at.
- **SQLite instead of JSON files.** Rejected for v1. Its strengths (queries across many Runs, multi-writer transactions, history) are for problems a single developer with a handful of Runs doesn't have. Its costs come on day one: a native module or a recent minimum Node version, and state you can't `cat`. The store sits behind a small load/save/delete interface, so this can change later.
- **Drain on shutdown**, finishing running Passes before exiting. Rejected. It can make Ctrl-C take up to 45 minutes, and it leaves the crash-recovery path running only when something has already gone wrong.

## Consequences

- **Every external write is check-then-act.** Each comment or reply sous-chef writes carries a step-tagged marker (for example `<!-- sous-chef:round=3 -->`), and sous-chef looks for it before writing. Other writes are naturally safe to repeat. The agent's output is saved locally *before* it is posted, so a retried step posts the same words.
- **An interrupted Pass stays the same Pass.** It resumes the same agent session, uses no extra retry or Feedback Round, and its wall-time clock carries on. Usage is saved as it arrives, so restarts can't reset or stretch a limit.
- **Lost local state is never guessed around.** If a Work Item is in flight but its record is missing or unreadable, it goes to Needs Human ("sous-chef lost this Run's local state"), and the developer re-Readies it to carry on.
- **The local file lives outside anything the agent can write** (`~/.local/state/sous-chef/<owner>/<repo>/`), so the agent can't edit its own counters.
- **Before anything resumes, sous-chef kills any agent process group** it saved that is still alive.
- **A per-repo lockfile refuses a second instance** on the same machine.

See [Where does Run state live, and how does sous-chef recover after a restart?](https://github.com/wkimdkim-nz/sous-chef/issues/11).
