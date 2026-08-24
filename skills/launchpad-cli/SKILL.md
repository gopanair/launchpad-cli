---
name: launchpad-cli
description: Deploy an app to a Launchpad install with `lp`, the command-line client — sign in without a browser, create an app, push the working directory, watch the build, read the logs, roll back, and run jobs and scheduled tasks. Load it whenever you are deploying to Launchpad, are asked to ship code to a Launchpad install, or hit an `lp` command that refused something. Nothing here needs Node, Python or a language toolchain on the machine you are working on — the install builds.
---

# Deploying to Launchpad with `lp`

`lp` pushes a directory to a Launchpad install, which builds it and serves it.
**The build happens on the install, not here.** You do not need `npm`, `node`,
`pip`, `uv`, a virtualenv or a Go toolchain on this machine to ship a Node,
Python, Go or static app — you need the source on disk and this one binary.

That is what makes the whole loop available to an agent: write files, `lp
deploy`, read what the platform says, fix, deploy again.

```bash
curl -fsSL https://raw.githubusercontent.com/gopanair/launchpad-cli/main/install.sh | sh
```

## Signing in without a browser

`lp login <url>` on its own opens a device flow and waits for a human. That is
the wrong door for an agent. Use a personal API key (`lpu_`) or a deploy key
(`lpd_`), which a person mints under **Account → API keys**:

```bash
printf '%s' "$LP_KEY" | lp login --token-stdin https://launchpad.corp
```

**Flags come before the positional argument.** `lp login <url> --token-stdin`
is refused, exit 2 — the flag would otherwise be dropped and you would be back
in the device flow.

There is no `--token` flag: a credential passed that way lands in shell history
and in every `ps` on the machine. The two other doors are environment variables:

| | |
|---|---|
| `LAUNCHPAD_TOKEN` | the key itself — **and it is only used when `LAUNCHPAD_URL` is set too** |
| `LAUNCHPAD_TOKEN_FILE` | a path to read it from, for a mounted secret |

A credential and an address are one statement. A token in the environment with
the install coming from anywhere else — a stored default, `--install` — is
refused, exit 2, naming both halves. Setting both variables at once is also a
refusal, not a precedence.

`lp login --status` says who and where. `lp login --forget <install>` drops a
stored credential; it does **not** revoke the key, and the command says so.

## The loop

```bash
cd ~/code/my-app
lp create --yes --link          # claim a slug, deploy this tree into it
# …edit files…
lp deploy                       # push what is on disk now
lp status                       # what is serving, since when, from what
lp logs --follow                # what the app said
lp events                       # what the platform noticed
lp rollback --yes               # back to the previous release
```

- **`lp create`** claims a name in the estate's namespace, so it says which slug
  it will take and asks first. `--yes` answers. `--link` writes
  `launchpad.toml`, so later commands in this directory know the app.
  `--repo <url>` creates from a repository instead and bundles nothing.
- **`lp deploy`** bundles the working directory as it is, uncommitted edits
  included. It needs no git remote, no commit, not even a repository. It never
  creates an app — a missing link is exit 2 naming `lp link <slug>`.
- **`lp redeploy`** rebuilds from the repository the app names and uploads
  nothing. On an uploaded app it is refused, naming `lp deploy`.
- **`lp restart --wait`** starts a stopped app and fails if it does not come up.
  `lp start` is the same call under the name that reads right when it is
  stopped; `lp stop` stops it.
- **`lp apps`** lists the apps this credential can see, `--mine` for the ones it
  owns. It is how to find a slug you do not know.

## Watching a build, and reading it afterwards

`lp deploy` prints the phases live — upload, scan, deploy — and exits non-zero
if the build fails. `--logs` interleaves the build log, which is where the
framework detection, the dependency install and the language version are
narrated. Add `--json` for NDJSON progress frames and one result object at the
end, which is the shape to parse rather than the pretty output.

Afterwards:

| | |
|---|---|
| `lp logs` | the app's runtime log — `--follow`, `--lines`, `--since 2h`, `--source app\|platform` |
| `lp logs --deploy <id>` | one build's log instead. `--source` and `--since` are refused with it |
| `lp events` | what the platform noticed going wrong — the command to run after a restart that did not take |
| `lp status --json` | the app, the serving release, and the app's URL |

## Exit codes

A contract, and the reason a failed run can be triaged without reading the log:

| | |
|---|---|
| 0 | it worked |
| 1 | the platform did the thing and it failed — a failed build, a refused scan, a failed job run |
| 2 | usage or configuration — no app linked, a bad flag, jobs not available on this install, a confirmation nothing could answer |
| 3 | not authenticated, or refused — no credential, a revoked or expired key, uploads switched off, a licence limit |
| 4 | the install did not answer |

## Running unattended

Confirmations are detected, not remembered: `lp` treats the run as unattended
when `--yes` is passed, `CI` is set, or stdin is not a terminal — and then it
**refuses rather than hanging** on a question nobody can answer. Pass `--yes` to
anything that would ask (`lp create`, `lp rollback`).

`-q` drops the narration and keeps the answer on stdout. `--json` is on every
command. `--no-color` (or `NO_COLOR`) for a clean log.

## What goes in the bundle

`.gitignore` decides — every one in the tree, `!` negations honoured — and git
is never shelled out to, so a plain folder bundles fine. `.launchpadignore` adds
to it. Always dropped: `.git/`, `.hg/`, `.svn/`, `node_modules/`, `venv/`,
`.venv/`, `__pycache__/`, `*.pyc`, `.DS_Store` — and `dist/`, `build/`, `out/`,
`.next/`, `.nuxt/`, `.svelte-kit/` **only when the tree root has a
`package.json`**, since a static site's `dist/` is the deliverable.

So: do not install dependencies before deploying. They would not travel, and the
install resolves them itself.

A bundled `.env`, `*.pem`, `id_rsa*` or `*.key` gets one dimmed warning and is
not refused — do not put secrets in the tree.

## Jobs and scheduled tasks

Two different things, and the CLI keeps them apart because the boundary is real:

- **`lp job list|run|runs|logs|cancel`** — a job runs in a container of its own,
  with its own memory limit and deadline, from the same release. Declare one in
  `launchpad.toml`. `lp job run <name>` waits by default and the run's outcome
  is the exit code; `--param k=v` repeats and is never parsed into a command.
  An install with no isolated backend has no jobs, and says so as exit 2.
- **`lp task list|run|runs|logs`** — a scheduled task is an HTTP request to a
  path your app published, in the app's own process. It takes no parameters, and
  a `skipped` firing is exit 1.

## Files: `lp store`

`lp store ls|put|get|rm` moves files in and out of an app's durable store from
outside the platform, for a person holding a grant. The transfer goes **direct
to the object store**, so an unreachable bucket is exit 4 naming that host,
while a refusal from the install is exit 3.

## What `lp` does not do

Be straight about this rather than inventing a flag:

| | |
|---|---|
| Environment variables | Not in the CLI. Set them on the app's Settings page, or `PATCH /api/v1/apps/{id}` with `env_vars`. A restart applies them; a redeploy is not required |
| Making an app public | Not in the CLI. `visibility` on the same PATCH, or the Settings page. A newly created app is private, so an anonymous fetch of its URL answers `authentication required` — that is the app working, not a broken deploy |
| Deleting an app | Not in the CLI |
| Grants, storage mappings, integrations | Administrator surfaces, in the UI or the API |

## When something refuses

| What you see | What it is |
|---|---|
| `no app is linked in this directory` | run `lp link <slug>`, or pass `--app` |
| `that key has expired` (exit 3) | mint another; nothing client-side decides this, the install said it |
| a licence limit (exit 3) | a limit on the install, not a permission problem, and nothing already deployed has stopped |
| the deploy failed during `scan` | the dependency gate, or the archive's caps — the phase and step are named, and `lp logs --deploy <id>` has the rest |
| `background jobs are not available` (exit 2) | this install has no isolated backend. Configuration, not a bug in the job |
| `authentication required` from the app's URL | the app is private. See the table above |

Everything takes `--help`.
