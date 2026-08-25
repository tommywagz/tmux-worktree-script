# tmux-worktree-script

`work` launches a squad of coding agents, each in its own git worktree on its own
branch, as tiled panes in a single tmux window — plus one agent in the repo root
on `main` that can merge the others' work.

Put the script wherever you like. `~/.local/bin/work` makes it available from any
repo on the machine; dropping it in a single project folder works too.

```bash
install -m 755 work ~/.local/bin/work
```

**Requirements:** `bash`, `git`, `tmux`, and either `python3` or `jq` (only needed
to read `opencode.json`). Agents themselves run `opencode`. A project that
configures the native Google Vertex provider also needs the Google Cloud CLI
(`gcloud`) and an interactive terminal for Google sign-in.

---

## Usage

```
work [--clean] [--no-root] <session_name> [num_worktrees] [base_branch]
```

| Argument | Default | Meaning |
| --- | --- | --- |
| `<session_name>` | *required* | tmux session name; also the worktree/branch prefix |
| `[num_worktrees]` | agent count, else `3` | how many **worktrees** to create (not panes — see below) |
| `[base_branch]` | `main` | branch the worktrees are cut from |

| Flag | Effect |
| --- | --- |
| `--clean` | Kill the session, remove its worktrees and delete its branches |
| `--no-root` | Give *every* agent a worktree; leave the repo root alone |

Flags work in any position, so `work squad --no-root` and `work --no-root squad`
are the same. `num_worktrees` must be an integer — a stray argument is rejected
rather than silently becoming a branch name.

| Environment variable | Effect |
| --- | --- |
| `WORK_DEFAULT_MODEL` | Model for any agent that doesn't name one. Unset = opencode's default |
| `WORK_AUTO_APPROVE` | `true`/`1`/`yes`/`on` adds `--auto` to the fallback roster's panes |

### Examples

```bash
work squad                          # squad from opencode.json or .agents/*.md
work new-feature 4 master           # 4 worktrees cut from 'master'
work squad --no-root                # no agent in the repo root
WORK_AUTO_APPROVE=true work squad   # auto-approve the fallback roster
work --clean squad                  # tear it all down
```

---

## What it builds

Running `work squad` in `~/code/myrepo` with four configured agents gives you:

```
~/code/
├── myrepo/                     <- pane 0: orchestrator, on main
│   └── jobs/                   <- the real shared-state directory
├── squad-worker-finder/        <- pane 1, branch agent-squad-finder
│   └── jobs -> ~/code/myrepo/jobs
├── squad-worker-creator/       <- pane 2, branch agent-squad-creator
│   └── jobs -> ~/code/myrepo/jobs
└── squad-worker-evaluator/     <- pane 3, branch agent-squad-evaluator
    └── jobs -> ~/code/myrepo/jobs
```

- Worktrees live **beside** the repo, at `../<session_name>-worker-<agent>/`.
- Branches are `agent-<session_name>-<agent>` — e.g. `agent-squad-finder`, not
  `finder`. Don't hardcode the short name anywhere.
- All panes share one tmux window named `workers`, tiled by alternating
  horizontal and vertical splits and re-tiled after each one, so a large squad
  doesn't run tmux out of space.
- Pane borders are labeled with the agent name. `opencode` likes to retitle its
  own pane, so the label is pinned per-pane to survive that.
- Pane 0 is focused when the session opens.

Re-running `work` for a session that already has worktrees **reuses** them and
ignores `num_worktrees`. If the tmux session is already alive, `work` runs any
required Vertex sign-in preflight and then just attaches — it does no provisioning
in that case, so kill the session first if you want the symlinks and job files
re-checked.

---

## The root agent: why one pane isn't in a worktree

An orchestrator's job is to merge worker branches into `main`. That is impossible
from a worktree, because the worktree has its own branch checked out — you cannot
merge into a branch you aren't on.

So one agent is designated the **root agent**: it gets no worktree and runs in the
repo root itself, on whatever branch is checked out there (normally `main`).

- **An agent named `orchestrator` becomes the root agent automatically.** No
  configuration needed.
- Any agent can opt in with `"worktree": false`, or out with `"worktree": true`.
- Root agents always take the **first pane(s)**, regardless of where they appear
  in the config, so pane 0 is always the one that can merge.
- `--no-root` overrides all of the above and gives everyone a worktree.
- Configuring two or more root agents is allowed but warned about — they would
  share one working tree and one branch and would overwrite each other.

Because the root agent occupies the repo root, **`num_worktrees` counts worktrees,
not panes**. Four agents with one root agent means 3 worktrees and 4 panes.

---

## Where the squad comes from

Three tiers, first match wins:

| Tier | Source | Behavior |
| --- | --- | --- |
| 1 | `opencode.json` with agent entries | Full control — models, prompts, auto-approve, placement |
| 2 | `.agents/*.md` | Each `<name>.md` becomes agent `<name>`, with that file as its first prompt |
| 3 | Neither | A lone `orchestrator` in the repo root, plus `num_worktrees` numbered worktrees running bare `opencode` |

In every tier, an agent named `orchestrator` lands in pane 0 in the repo root — so
the first pane is the one that can merge, with or without any config file.

### Tier 2: zero-config from `.agents/*.md`

If a repo already holds `.agents/orchestrator.md`, `.agents/finder.md`, and so on,
that *is* the roster. No `opencode.json` required:

- Only `.md` files sitting **directly** in `.agents/` count. Subdirectories such
  as `.agents/skills/` are ignored, since those hold skill definitions rather than
  agent prompts.
- Workers are ordered alphabetically. Use `opencode.json` if you want a specific
  pane order.
- Models come from `WORK_DEFAULT_MODEL`, auto-approve from `WORK_AUTO_APPROVE`.
- `.agents/` is looked for in the invocation directory, then the repo root.

---

## Tier 1: `opencode.json`

On start, `work` looks for `opencode.json` in the directory you run it from,
falling back to the repo root.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "agents": {
    "orchestrator": {
      "model": "google/gemini-3.5-flash",
      "systemPromptFile": ".agents/orchestrator.md",
      "autoApprove": true,
      "worktree": false
    },
    "finder": {
      "model": "google/gemini-2.5-flash",
      "systemPromptFile": ".agents/finder.md",
      "autoApprove": true
    }
  }
}
```

| Key | Effect |
| --- | --- |
| `model` | Passed as `opencode --model <model>`. Must be a real id — check with `opencode models` |
| `systemPromptFile` | Its **contents** are passed as `--prompt`, so the file is the first message that agent receives. Resolved relative to `opencode.json` |
| `autoApprove` | `true` appends `--auto`, so that agent runs without permission prompts |
| `worktree` | `false` puts the agent in the repo root instead of a worktree |

For each agent, in the order the entries appear:

- the name becomes the worktree branch (`agent-<session>-<name>`), its directory
  (`../<session>-worker-<name>`) and its pane label
- the pane runs `opencode --model <model> --auto --prompt "<file contents>"`,
  with each piece included only if configured

Details:

- Entries may sit under `"agents"`, under `"agent"`, or at the top level of the
  file.
- A top-level `"autoApprove"` sets the default for every agent that doesn't state
  its own, so you can flip the whole squad in one place.
- Any one of `model`, `systemPromptFile`, `autoApprove` or `worktree` is enough to
  make an entry count as an agent. An entry with only `"autoApprove": true` still
  gets a pane, running plain `opencode --auto`.
- A `systemPromptFile` that is missing or empty just means that pane starts with
  no first prompt — the pane still launches.
- Agent names may contain only letters, digits, `.`, `_` and `-`, since the name
  becomes a branch and a directory. Others are skipped with a warning.
- `num_worktrees` defaults to the number of agents that want one, so `work squad`
  alone gives one pane per agent.
- Ask for more worktrees than you have agents and the extras fall back to the
  pre-config behavior: a branch named after their **1-based worktree position**,
  running plain `opencode`, labeled with their directory name. With one worktree
  agent and `num_worktrees 4` you get `agent-squad-finder`, then `agent-squad-2`,
  `agent-squad-3`, `agent-squad-4`.
- Ask for fewer and only the first N worktree agents are used.

> **`agents` is a `work` key, not an opencode key.** opencode's own config uses
> `agent` (singular) with different sub-keys, and it silently ignores `agents`
> entirely — confirm with `opencode debug config`. That separation is deliberate:
> `work` translates these entries into CLI flags, so the two never fight over the
> same field. The tradeoff is that opencode itself gets no agent definitions from
> this file.

### Google Vertex AI

When the selected project `opencode.json` configures Google Vertex through a
native provider class, `work` runs `gcloud auth application-default login`
**before** it creates/attaches a tmux session or creates/reuses worktrees. If the
provider declares `options.project`, as in the example below, the script passes
that value as `--project` so ADC quota and billing target the same Cloud project
as the Vertex requests. Finish the browser sign-in and return to the terminal;
cancelling or failing it leaves the repository and tmux unchanged.

For the OpenCode v1-style config used by this script's examples, the provider
looks like this (the provider ID, `vertex` here, is your choice):

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "vertex": {
      "npm": "@ai-sdk/google-vertex",
      "name": "Google Vertex AI",
      "options": {
        "project": "my-gcp-project",
        "location": "global"
      }
    }
  },
  "agents": {
    "orchestrator": {
      "model": "vertex/gemini-2.5-pro",
      "worktree": false
    }
  }
}
```

OpenCode v2 calls the corresponding fields `providers`, `package`, and
`settings`; `work` recognizes both forms, including the native
`@opencode-ai/ai/providers/google-vertex` package and its transport variants.
It deliberately detects the provider **class**, not the provider ID, so a custom
name such as `company-vertex` works too.

Vertex also needs a Google Cloud project with the Vertex AI API enabled. Supply
the project and optional location in the provider settings as above, or export
`GOOGLE_VERTEX_PROJECT` and (optionally) `GOOGLE_VERTEX_LOCATION` in the shell
profile used by tmux. `location: "global"` / `GOOGLE_VERTEX_LOCATION=global` is
the AI SDK default; use a regional location when data residency requires one.

---

## How the panes authenticate

For ordinary API-key providers, **`work` never touches credentials.** There is
no key handling anywhere in the script — no environment variable is set, no
config is written, and nothing is typed into a pane but the `opencode` command
line itself. The one exception is a detected native Vertex provider: `work`
starts the interactive `gcloud auth application-default login` preflight
described above. Google writes the resulting Application Default Credentials;
`work` does not read or write them.

Every pane then authenticates as the same Unix user. OpenCode finds regular
provider credentials from sources that are per-user and machine-wide rather than
per-pane:

| Source | Where | Written by |
| --- | --- | --- |
| Credential store | `~/.local/share/opencode/auth.json` (mode `0600`) | `opencode auth login` |
| Environment | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, … | your shell profile |
| Vertex ADC | Google Cloud's Application Default Credentials | `gcloud auth application-default login --project=<configured-project>` |

Both are visible to every pane at once, which is the whole reason a squad needs
no per-agent setup: pane 3 is the same Unix user on the same machine as pane 0,
so it reads the same `auth.json` and inherits the same environment. Adding a
fifth agent costs nothing in credential terms.

### How the environment actually reaches a pane

`work` starts each agent with `tmux send-keys`, which *types* the command into a
pane that is already running a shell — it does not `exec opencode` directly. So
the chain is:

```
tmux split-window  ->  login bash  ->  ~/.profile  ->  ~/.bashrc  ->  export *_API_KEY
                                                                          |
                       tmux send-keys "opencode --model … --prompt …" ----+--> opencode
```

tmux runs `default-shell` as a **login** shell (its `default-command` is empty),
so each pane sources your profile from scratch before the sent command runs.
That has a useful consequence: a key you add to `~/.bashrc` is picked up by the
next pane you open, with no need to restart the tmux server.

Confirm what any given pane will see:

```bash
opencode auth list    # prints stored credentials and detected env vars separately
```

### Caveats

- **A key exported only in your current shell may not reach the panes.** The
  tmux *server* captures its environment when it first starts, and refreshes
  only the variables in `update-environment` (`DISPLAY`, `SSH_AUTH_SOCK` and
  friends — no API keys) for later sessions. A key that lives in your profile is
  fine, because the pane's login shell re-reads it. A one-off `export
  FOO_API_KEY=…` before running `work` is not: kill the server (`tmux kill-server`)
  or put it in the profile.
- **Don't inline a key into the pane command.** Because agents are launched with
  `send-keys`, anything you prepend lands in that pane's visible scrollback *and*
  its shell history. Use `opencode auth login` or the profile instead.
- **`work` cannot give two panes different credentials.** Model is per-agent
  (`"model"` in `opencode.json`), identity is not. Every pane runs as you.
- **Vertex sign-in is intentionally a launch preflight.** If the selected config
  uses one of the recognized native Vertex classes, `work` asks gcloud to sign in
  even if a tmux session of that name already exists. Use `work --clean` to tear
  a session down without signing in; cleanup never launches an agent.
- If a provider is configured in *both* `auth.json` and the environment,
  resolving that is opencode's business, not `work`'s. Keep one to avoid
  wondering which won.

---

## Shared job state

Git worktrees do not share working files. A file one agent writes in its worktree
is invisible to the others until it is committed and merged — far too slow for
coordination.

So `work` maintains exactly **one** physical `jobs/` directory, in the repo root,
and drops a `jobs` symlink into every worktree pointing at it. Every pane then
reads and writes shared state through the plain relative path `jobs/...`, and a
write from any pane is visible to all of them instantly.

What gets provisioned (existing files are **never** overwritten):

```
jobs/
├── backlog.json          seeded empty if absent
├── inbox/<agent>.json    one per configured agent — orchestrator writes, that agent reads
└── status/<agent>.json   one per configured agent — that agent writes, everyone reads
```

The one-writer-per-file split is deliberate: whole-file JSON writes are not atomic
against each other, so a single shared `status.json` written by four agents would
silently lose updates.

### Add `/jobs` to `.gitignore` — with no trailing slash

```gitignore
/jobs
```

In a worktree, `jobs` is a **symlink**, which git treats as a file. A directory-only
pattern (`/jobs/`) would not match it, leaving the symlink showing up as untracked
in every worktree — and an orchestrator that merges branches needs a clean
`git status`. `work` checks this with `git check-ignore` inside a real worktree and
warns if it isn't set up.

Two consequences of `jobs/` being untracked:

- Coordination traffic never causes a merge conflict, and never dirties the tree.
- It does not survive a fresh clone. `work` seeds an empty `backlog.json` so an
  orchestrator doesn't trip over a missing file.

`--clean` deliberately **leaves `jobs/` in place**, so a backlog and the progress
recorded against it survive tearing the session down and bringing it back up.

---

## Cleanup

```bash
work --clean <session_name>
```

Kills the tmux session, force-removes every worktree belonging to that session,
deletes its `agent-<session>-*` branches, and prunes stale worktree records.
Worktrees are found both by branch prefix and by directory name, so ones left in
an older layout are still cleaned up. The shared `jobs/` directory is preserved.

---

## Warnings you may see

| Warning | What to do |
| --- | --- |
| `'jobs' is not gitignored on branch 'main'` | Add `/jobs` to `.gitignore` and **commit it** — worktrees are checked out from the last commit, not your working tree |
| `the repo root is on 'X', not 'main'` | The root agent merges into whatever is checked out there. `git checkout main` first |
| `the repo root has uncommitted changes` | The orchestrator needs a clean tree to merge, and workers only see committed work |
| `N agents are configured without a worktree` | Give all but one a worktree — they'd otherwise share a branch and clobber each other |
| `no agent runs in the repo root` | Name an agent `orchestrator`, or set `"worktree": false` on one, so something can merge |
| `<dir>/jobs exists and is not a symlink` | A real `jobs/` in a worktree shadows the shared one. Remove it; that pane can't see shared state |
| `Skipping stale worktree <session>-worker-orchestrator` | Left over from before that agent moved to the repo root. Remove it with the printed command |

---

## Gotchas

- **Commit before you launch.** Worktrees are created from the last *commit* on
  `base_branch`. Uncommitted changes in the root are invisible to the workers.
  `systemPromptFile` is the exception — it's read from the config directory at
  launch, so prompt edits take effect without committing.
- **An already-running session short-circuits everything.** `work` attaches and
  skips provisioning. Kill it first to re-run setup.
- **Existing worktrees win over `num_worktrees`.** Use `--clean` to change the
  squad size.
- **Model ids are not validated by `work`.** A bad id fails inside that pane at
  launch. Check against `opencode models` first.
- **`work` only starts Vertex ADC sign-in.** If a Vertex pane still complains
  about authentication, confirm its Cloud project, enabled Vertex AI API, and
  selected model; for other providers, see
  [How the panes authenticate](#how-the-panes-authenticate).
- **`--auto` is genuinely dangerous.** opencode describes it as auto-approving
  every permission not explicitly denied. Fine for a sandboxed repo you can throw
  away; think twice anywhere else.
