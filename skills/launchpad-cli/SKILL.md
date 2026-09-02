---
name: launchpad-cli
description: Deploy an app to a Launchpad install with `lp`, the command-line client — sign in without a browser, create an app, push the working directory, watch the build, read the logs, roll back, and run jobs and scheduled tasks. Load it whenever you are deploying to Launchpad, are asked to ship code to a Launchpad install, or hit an `lp` command that refused something. Nothing here needs Node, Python or a language toolchain on the machine you are working on — the install builds.
---

# Deploying to Launchpad with `lp`

`lp` pushes a directory to a Launchpad install, which builds it and serves it.
**The build happens on the install, not here.** You do not need `npm`, `node`,
`pip`, `uv`, a virtualenv, a Go toolchain or an R installation on this machine
to ship a Node, Python, Go, R or static app — you need the source on disk and
this one binary.

That is what makes the whole loop available to an agent: write files, `lp
deploy`, read what the platform says, fix, deploy again.

| Platform | How |
|---|---|
| Linux x86_64, arm64 | `install.sh` |
| macOS Intel, Apple Silicon | `install.sh` |
| Windows x86_64 | download `lp_windows_amd64.exe` by hand |

On Linux and macOS:

```bash
curl -fsSL https://raw.githubusercontent.com/gopanair/launchpad-cli/main/install.sh | sh
```

On **Windows** the script refuses rather than downloading something that does
not exist. Take the binary directly and put it on your PATH:

```powershell
curl.exe -fsSLo lp.exe https://github.com/gopanair/launchpad-cli/releases/latest/download/lp_windows_amd64.exe
```

**The binaries are unsigned, and on Windows that can stop the install dead.**
With Smart App Control enforcing, `lp.exe` is refused however it was obtained —
the download is intact and matches the published checksum, and Windows blocks it
anyway for having no signature and no reputation. There is no per-file
exception: the only lever is switching Smart App Control off, which Windows
cannot reverse without a reinstall. **Do not talk anybody into that.** Report
that the install is blocked and say the fix is a signed build.

On macOS the same lack of a signature is survivable, because Gatekeeper
quarantines what a browser downloads and not what `curl` does — so the command
above works, while a binary fetched by hand from the releases page has to be
cleared before it will run.

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

**A key has a scope, and a read-scoped one cannot deploy.** The scope is fixed
when the key is minted and cannot be widened afterwards, so a read key is
replaced rather than changed. The key `lp login`'s device flow mints is always
full — there is no `--scope` — so a read key can only reach you through
`--token-stdin` or the environment, from a person who minted one in the browser.
Every GET works, and the first write comes back exit 3 with
`code: "key_read_scope"`. Ask for a full key; there is nothing to retry.

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
lp deps                         # what the dependency policy says about it
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

**Read `reason` before you read anything else.** A failed deploy's result object
carries the whole failure message, a `reason` key, and the detail that goes with
that reason — so in most cases you do not need a second command:

```json
{"deployment_id":"…","app":"my-app","status":"failed","phase":"scan","step":"scan",
 "reason":"dependency_blocked","message":"REFUSED: …","dependencies":{"…":"…"}}
```

| `reason` | What it means, and what to do |
|---|---|
| `build_failed` | The build or dependency install failed. `log_tail` is in the object; `lp logs --deploy <id>` has the rest |
| `start_failed` · `not_ready` | It **built** and then would not answer. The build log ends in "Ready." and tells you nothing — read `lp logs`, the app's own output |
| `dependency_blocked` | This install's dependency policy refused named packages. The findings are in the object under `dependencies`, and `lp deps` is the same answer. **Not a bug in your code** |
| `source_blocked` | The install refuses that repository, URL or branch. Not fixable by pushing; ask an administrator |
| `unsupported_framework` | Nothing in the tree the platform knows how to serve |
| `manifest_invalid` | `launchpad.toml` was read and rejected. A one-file fix, and the message says which |
| `runtime_version` | The language version you declared is not available on this install |
| `source_unavailable` | The clone, the credential, or a pruned rollback target — the code could not be fetched |
| `internal_error` | The platform, not your app. Retry, then tell somebody |

A `reason` you do not recognise means the install is newer than your `lp`; treat
it as `build_failed` and read the build log. An **absent** `reason` means the
install is older than the field — it is not a category, so fall back to `phase`
and `step`.

Afterwards:

| | |
|---|---|
| `lp logs` | the app's runtime log — `--follow`, `--lines`, `--since 2h`, `--source app\|platform`. `--json` is one object per line: `at`, `source`, `stream`, `line`, and on exactly one line a `marker` |
| `lp logs --deploy <id>` | one build's log instead. `--source` and `--since` are refused with it |
| `lp events` | what the platform noticed going wrong — the command to run after a restart that did not take |
| `lp status --json` | the app, the serving release, and the app's URL |
| `lp deps --json` | the packages, the advisories, and the versions that fix a refused deploy |

## When the dependency policy refuses a deploy

Some installs will not deploy packages with known advisories. That is an
administrator's decision about the whole install, not a judgement about your
code, and **it is the one failure with a structured answer waiting for you**:

```sh
lp deps --json | jq '.refused.findings[] | {package, version, advisory_id, severity, fixed_in, path}'
```

- `fixed_in` is the version to move to. When it is empty, no fix has been
  published and upgrading is not the route.
- `path` is the chain — `my-app > webpack > lodash`. When a finding is not
  `direct`, upgrading the package it names is not something your manifest can
  do; the chain says which of *your* dependencies to bump instead.
- `mode` and `enforcing` say whether this install is actually blocking. In
  `warn` mode findings are recorded and nothing is refused.
- A waiver is per advisory, expires, and only an administrator can grant one.
  There is no permanent waiver and nothing you can do from here to bypass the
  gate — if there is no fixed version, the routes are dropping the package or
  asking an administrator.

- When an administrator grants a waiver, or the advisory feed moves,
  `POST /api/v1/apps/{id}/dependency-recheck` re-derives this app's findings
  without a deploy. It resolves nothing and installs nothing — same packages,
  same feed — so it is worth a call only when one of those two has changed.

`lp deps` needs edit access to the app: an exact package-and-version list of a
running internal tool is a target map, so a viewer cannot read it.

## Looking at the app

`lp status --json` carries `url`, and it is the install's own answer rather than
its address with a path stuck on the end — so it is right whether or not that
install serves deployed apps on a second origin.

A new app is private, and private means the same thing to a script as to a
browser: an anonymous GET answers `authentication required`. **Your own key is
the way in.** The apps gateway takes it as a bearer credential and then
authorizes you exactly as it would in a browser:

```sh
URL=$(lp status --json | jq -r .url)
curl -fsS -H "Authorization: Bearer $LAUNCHPAD_TOKEN" "$URL/"
```

The gateway consumes the key and strips it: the app process never sees it, and
never learns which credential the request arrived on. An app key (`lp_…`) works
the same way for the one app it grants.

**In a headless browser** — for a screenshot, or a page that only exists after
its JavaScript has run — send the same header on every request the page makes:

```js
// Playwright
const ctx = await browser.newContext({
  extraHTTPHeaders: { Authorization: `Bearer ${process.env.LAUNCHPAD_TOKEN}` },
});
// Puppeteer:  await page.setExtraHTTPHeaders({ Authorization: ... })
// CDP direct: Network.setExtraHTTPHeaders
```

It has to reach the subresources and not just the document, or the CSS and the
scripts come back refused and the app renders as broken rather than as private
— which is why the header goes on the context or the page, not on one
navigation. The same breadth is the one thing to be careful with: these APIs add
the header to *every* host the page touches, so a page that pulls a font or a
script from somewhere else would send your key there too. If the app is not
self-contained, scope it with a request interceptor.


## Exit codes

A contract, and the reason a failed run can be triaged without reading the log:

| | |
|---|---|
| 0 | it worked |
| 1 | the platform did the thing and it failed — a failed build, a refused scan, a failed job run |
| 2 | usage or configuration — no app linked, a bad flag, jobs not available on this install, a confirmation nothing could answer, **an app that is locked**, or **a name that does not resolve** |
| 3 | not authenticated, or refused — no credential, a revoked or expired key, uploads switched off, a licence limit |
| 4 | the install did not answer |

## Running unattended

Confirmations are detected, not remembered: `lp` treats the run as unattended
when `--yes` is passed, `CI` is set, or stdin is not a terminal — and then it
**refuses rather than hanging** on a question nobody can answer. Pass `--yes` to
anything that would ask (`lp create`, `lp rollback`, `lp task delete`).

`-q` drops the narration and keeps the answer on stdout. `--json` is on every
command. `--no-color` (or `NO_COLOR`) for a clean log.

**A refusal is JSON too.** Under `--json` anything that exits non-zero without
having already written a result object writes one instead of leaving you to parse
English off stderr:

```json
{"error":"that key has expired","kind":"auth","code":"key_expired","exit":3}
```

`kind` is the client's classification — `usage`, `auth`, `unreachable`, `locked`,
`failed` — and `code` is the install's own, present only when it sent one. The
sentence still goes to stderr as well. A **failed deploy is the exception**: its
result object already said everything, so there is no second document.

## What goes in the bundle

`.gitignore` decides — every one in the tree, `!` negations honoured — and git
is never shelled out to, so a plain folder bundles fine. `.launchpadignore` adds
to it. Always dropped: `.git/`, `.hg/`, `.svn/`, `node_modules/`, `venv/`,
`.venv/`, `__pycache__/`, `*.pyc`, `.DS_Store` — and `dist/`, `build/`, `out/`,
`.next/`, `.nuxt/`, `.svelte-kit/` **only when the tree root has a
`package.json`**, since a static site's `dist/` is the deliverable.

So: do not install dependencies before deploying. They would not travel, and the
install resolves them itself.

**R is the one where that costs real megabytes.** A local `renv::restore()`
fills `renv/library/` with the whole dependency tree, and nothing above drops it
by default — an R tree has no `package.json`, so the `dist/`-style rules do not
apply. Ignore `renv/library/` and `renv/staging/` (the usual R `.gitignore`
does) and ship `renv.lock` alone; the install restores from the lockfile against
its own binary repository, and a library built on your machine would be compiled
against your distribution rather than the install's.

A bundled `.env`, `*.pem`, `id_rsa*` or `*.key` gets one dimmed warning and is
not refused — do not put secrets in the tree.

## Jobs and scheduled tasks

Two different things, and the CLI keeps them apart because the boundary is real:

- **`lp job list|run|runs|logs|cancel`** — a job runs in a container of its own,
  with its own memory limit and deadline, from the same release. Declare one in
  `launchpad.toml`. `lp job run <name>` waits by default and the run's outcome
  is the exit code; `--param k=v` repeats and is never parsed into a command.
  An install with no isolated backend has no jobs, and says so as exit 2.
- **`lp task list|create|edit|delete|run|runs|logs`** — a scheduled task is an
  HTTP request to a path your app published, in the app's own process.
  `lp task run <name>` fires one now and waits as the job verb does, and a
  `skipped` firing is exit 1. It takes no parameters *here*: a task's parameters belong to
  its schedule, because a request to a published path has nowhere else to put
  them.

  `lp task create <name> --schedule "0 9 * * *" --path /digest` is the whole of
  it; `--method`, `--overlap` and `--enabled` are optional. **`lp task edit`
  sends only the fields you name**, so naming one leaves the rest alone.
  `lp task delete` asks first, because a task takes its run history with it —
  pass `--yes` where nothing can answer.

**The two are created in different places, and only one of them here.** A job is
declared in `launchpad.toml` and exists once the release is built — and a
`schedule` on a job there makes it a *scheduled job*, still a container of its
own, not a task. A task is a row on the app, and `lp task create` is the way to
make one. A full personal key reaches these, a **deploy key does not** — shaping
a task is configuration rather than shipping.

**A `[tasks]` table in `launchpad.toml` does nothing.** Only `[jobs]` is read
there. An unrecognized table is ignored rather than refused, because the file is
shared with `[app]` and `[static]` — but the build log now says so, so check it
rather than assuming a silent deploy means a schedule exists.

## Files: `lp store`

`lp store ls|put|get|rm` moves files in and out of an app's durable store from
outside the platform, for a person holding a grant. The transfer goes **direct
to the object store**, so an unreachable bucket is exit 4 naming that host,
while a refusal from the install is exit 3.

## The SDK before the deploy: `lp sdk vendor`

The app SDK is installed by the build, so a deployed app never needs this. What
it is for is everything before the deploy — an editor that can see the package,
and a local `go build`, `go vet` or CI run that does not fail on the import.

```
lp sdk vendor                  # language detected from the tree
lp sdk vendor --lang python
```

It writes `.launchpad/sdk-<lang>/` from the install you are signed in to, so the
bytes you compile against are the ones that install's build would have used.
Nothing is published to a registry in any language, and that is deliberate: a
published package is a second version, and you would then be building against
whichever one your package manager resolved.

- **Go** also gets a `go.work` naming the vendored module. No require line is
  added to your `go.mod` — a workspace resolves the import without one. Both
  paths ship with `lp deploy` and the platform then stands aside, because an app
  that brought its own SDK keeps it. **Ignore both in `.gitignore` or neither**:
  ignoring only `go.work` leaves a workspace on the platform naming a directory
  that did not ship, and the build fails there while working here.
- **Python**: `uv pip install -e .launchpad/sdk-python`.
- **Node**: `"@launchpad/sdk": "file:.launchpad/sdk-node"` in `package.json`.
- **R**: `renv::install(".launchpad/sdk-r")`, or nothing at all — the build
  installs `launchpad` into the release's library either way, and unlike Go
  there is no compile step here to fail without it.

Re-run it after the install is upgraded. Delete both paths to go back to the
platform's own copy.

## The escape hatch: `lp api`

`lp api <method> <path>` makes an authenticated call with the credential `lp`
already holds, and prints the install's own JSON on stdout.

```bash
lp api GET /api/v1/apps
lp api PATCH /api/v1/apps/$ID --data '{"visibility":"public"}'
lp api POST /api/v1/apps/$ID/deployments --data @body.json
cat body.json | lp api POST /api/v1/apps/$ID/cron --data @-
```

**Never read the credential out of the config file.** There is no command that
prints it, and that is deliberate — its path and format are an implementation
detail, they differ per platform, and a regex over them breaks silently the day
the format changes. `lp api` is how you make the call without ever holding the
key: it keeps the secret out of your shell variables, out of the process table,
and out of whatever captured your output.

Exit codes are the usual contract, so a refusal is still 2, 3 or 4 rather than
something to parse out of the body. A path with no leading slash is read as
relative to `/api/v1/`.

**Prefer a verb where one exists.** `lp api` validates nothing but your JSON: a
real verb checks the arguments, names the one you left out, reports failures in
sentences, and keeps working when the route behind it changes. Reach for `lp
api` for the rows below, not instead of `lp deploy`.

## What `lp` does not do

Be straight about this rather than inventing a flag. Everything here is reachable
with `lp api`, which is the point of the column on the right:

| | |
|---|---|
| Environment variables | Not in the CLI. Set them on the app's Settings page, or `PATCH /api/v1/apps/{id}` with `env_vars`. A restart applies them; a redeploy is not required |
| Making an app public | Not in the CLI. `visibility` on the same PATCH, or the Settings page. A newly created app is private, so an *anonymous* fetch of its URL answers `authentication required` — that is the app working, not a broken deploy, and your own key fetches it (see Looking at the app) |
| Deleting an app | Not in the CLI |
| Creating, editing or deleting a scheduled task | `lp task create\|edit\|delete`, with a full personal key |
| A notebook's re-render schedule | Not in the CLI, and not a scheduled task — it produces a new rendered release rather than calling a path. The Automation tab, or `PUT`/`DELETE /api/v1/apps/{id}/render-schedule` |
| An app's tags | Not in the CLI. `PUT /api/v1/apps/{id}/tags` with the ids of tags that already exist; creating the tags themselves is an administrator's |
| Pruning old releases | Not in the CLI. `DELETE /api/v1/apps/{id}/deployments` drops every release but the serving one, and `DELETE .../deployments/{deployID}` drops one. A pruned release is no longer a rollback target |
| Grants, storage mappings, integrations | Administrator surfaces, in the UI or the API |

## When something refuses

| What you see | What it is |
|---|---|
| `no app is linked in this directory` | run `lp link <slug>`, or pass `--app` |
| `that key has expired` (exit 3) | mint another; nothing client-side decides this, the install said it |
| `that key can read and not write` (exit 3, `key_read_scope`) | the key was minted read-only in the browser. A scope cannot be widened — ask for a full key. Retrying, or reading first to "warm it up", changes nothing |
| `forbidden` (exit 3, `deny_system_role`) | the account is a **viewer** on this install and publishing needs the publisher role. The install says only "forbidden" — this code is the whole diagnosis. An administrator changes a system role; there is no flag, no other command and no retry that reaches it. A **personal** key can lodge the ask — an empty `POST /api/v1/me/promotion-requests`, which notifies the administrators and answers 409 if one is already pending — and then say who has to act. A deploy key cannot, and neither can a service account |
| a licence limit (exit 3) | a limit on the install, not a permission problem, and nothing already deployed has stopped |
| the deploy failed during `scan` | the dependency gate, or the archive's caps. `reason` says which, and `lp deps` has the packages |
| an app that is locked (exit 2) | an administrator, or the install's own dependency policy, has locked it. **Nothing was built**, and retrying will refuse identically until the lock is lifted |
| a task or job name that does not resolve (exit 2) | a name, not a run — the platform attempted nothing. `lp task list` and `lp job list` name what exists. This is exit **2**, never 1: exit 1 would mean the work ran and failed |
| `background jobs are not available` (exit 2) | this install has no isolated backend. Configuration, not a bug in the job |
| `authentication required` from the app's URL | the app is private and the request carried no credential — send your key, per Looking at the app |

Everything takes `--help`.
