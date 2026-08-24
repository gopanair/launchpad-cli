---
name: launchpad-sdk
description: Write an app that runs on Launchpad — what the platform sets in its environment, how it is detected and built, what `launchpad.toml` declares, and the Launchpad app SDK for Python and Node (`lp.app.can`, scratch and storage, the app's JSON document, Slack and email, background jobs, who the viewer is, a scheduled task's log). Load it when writing or fixing code that will be deployed to Launchpad, when an app works locally and misbehaves once deployed, or when an app needs to send a notification, store a file, or start background work.
---

# Writing an app for Launchpad

An app is ordinary code. Launchpad clones or receives it, works out what it is,
installs its dependencies, starts it, and puts it behind a URL. There is nothing
to import to be deployable — the SDK below is for the things only the platform
can do.

## The contract every app keeps

| | |
|---|---|
| `PORT` | listen on it. Never on a port you chose |
| `HOST` | bind it. `app.listen(port)` alone binds `0.0.0.0`, which is reachable without going through the platform |
| `BASE_PATH` | where you are mounted, e.g. `/apps/reports`. Empty on an install that gives apps their own origin |

**Links and form actions carry `BASE_PATH`; redirects must not.** The proxy
strips the prefix before your handler sees the request and re-adds it on the way
out, so a `Location: /done` is right and `Location: /apps/reports/done` is
doubled. Root-absolute asset URLs (`/styles.css`) ask the *platform*, not you,
and 404.

Anything else you need — a database URL, an API key — is an environment variable
someone sets on the app's Settings page. A restart applies it; a redeploy is not
required. Your process never receives the platform's own secrets.

## How the build decides what your app is

Detected from the tree, in this order:

| | |
|---|---|
| **Next.js** | `next.config.*` or `next` in `package.json`. A `build` script is required; do **not** set `basePath` or `output` |
| **Node** | any `package.json` with a `start` script. A `build` script runs first if present |
| **Flask / FastAPI / Streamlit** | `flask` / `fastapi` / `streamlit` in `requirements.txt` |
| **Generic Python** | `requirements.txt` with no framework recognised |
| **Go** | `go.mod` plus a `main` package — module root, else the first `cmd/*`. Built into one binary |
| **Static** | last, and only when the tree has none of `package.json`, `requirements.txt`, `pyproject.toml`, `Pipfile`, `go.mod`. A `[static]` table overrides that |

Traps worth knowing before you hit them:

- **Never set `NODE_ENV=production` as an app variable.** It reaches the build,
  npm reads it as `--omit=dev`, and the build then fails on a missing dev
  dependency (`vite: command not found`). The platform sets it for the *running*
  process itself.
- **Dependency install scripts do not run** (`--ignore-scripts`), so anything
  that depended on a `postinstall` — `prisma generate`, a native rebuild —
  belongs in your `build` script.
- **npm only** for Node. A committed `package-lock.json` selects `npm ci` and
  must be in sync; a yarn or pnpm lockfile alone means versions resolve fresh at
  deploy time.
- **A Go app cannot run an npm build during deploy.** Commit the built assets.
- **A frontend-only repo with a `package.json`** is not static. Either add a
  server and a `start` script, or commit the build output and declare
  `[static] root = "dist"`.
- The language version is **discovered on the install, never approximated**. A
  pin naming a patch version is refused rather than truncated; declare it once
  in the file the language already uses (`go.mod` for Go).

## `launchpad.toml`

Optional, committed, and the better door for anything it can say. Unknown
top-level tables are ignored, so it is safe to share with the CLI's `[app]`.

```toml
[app]
slug = "reports"              # written by `lp link`

[static]
root     = "dist"             # document root
entry    = "report.html"      # what the app's URL serves
fallback = true               # client-side routing

[jobs.nightly-report]
command  = "python -m reports.nightly"
schedule = "0 3 * * *"        # optional; omit for a job you start yourself
timeout  = 1800
memory   = 1024
retries  = 2                  # off by default
viewer_runnable = true        # a viewer may start it

[jobs.nightly-report.params.region]
type     = "string"
required = true
```

A job declared here versions with the code it runs, and a rollback takes the
command back with it. Static apps may not declare jobs — there is no runtime in
which a command would mean anything — but scheduled tasks are fine, since those
are HTTP requests to a path you published.

## The SDK

**Installed by the build, from inside the platform binary.** There is no
registry fetch and nothing to add to `requirements.txt` or `package.json`; if
you declare your own copy, yours is kept.

```python
import launchpad as lp        # Python
```

```js
import lp from "@launchpad/sdk";   // Node — ESM. From CommonJS: await import(...)
```

Off-platform every call that needs the platform raises or throws, naming the
environment variable that is missing. **There is deliberately no dry-run mode**:
an SDK that printed to a console instead of paging a channel would be a trap
that only springs in production.

### What this app is, and what it may do

```python
lp.app.id, lp.app.slug, lp.app.mode, lp.app.base_path
lp.app.can("email")            # cheap and cached
```

```js
await lp.app.slug();  await lp.app.can("slack");
```

An administrator attaches an integration to an app; the app never holds the
Slack token or the mail credential and never names the channel. **Ask before you
send** — an app that checks at startup can tell someone "email is not enabled
for this app" on a screen instead of leaving it to be found at 2am in a log.

### Files

| | |
|---|---|
| `lp.scratch_dir("uploads")` / `lp.scratchDir` | ephemeral, per app, **wiped on every start, restart and redeploy** |
| `lp.session_scratch_dir(id)` / `lp.sessionScratchDir` | one browser session's corner of it. The id is hashed, so a traversing value cannot escape; an empty one raises |
| `lp.storage("reports").path` | a durable volume an administrator mapped to this app, resolved by name |
| `lp.store("reports").put(path)` | a file into an object store |
| `lp.mounts()` | everything mapped, with `read_only` |

### The app's one document

```python
lp.data.get()                                   # dict, or None if never written
lp.data.set({"period": period})
lp.data.update(lambda s: s.update(runs=s.get("runs", 0) + 1))   # against a second writer
```

One JSON document per app, 256 KiB, no keys and no queries — it survives
restart, redeploy and rollback. Never-written, cleared and `{}` stay tellable
apart. It is a place for a little state, not a database.

### Telling someone

```python
lp.notify.slack("Ingest finished: 1,204 rows")
lp.notify.slack_file("/tmp/recon.xlsx", title="Weekly reconciliation")
lp.notify.email(to=["finance@example.com"], subject="Weekly report", text=body)
send = lp.notify.status(send)
```

Nothing here blocks on delivery: a send returns once the platform has accepted
and recorded it, and the record — not the message — is what a workflow may rely
on. A send never fails the request that triggered it.

### Background work

```python
run = lp.jobs.start("reconcile", params={"month": "2026-03"})   # as the app
run = run.wait(timeout=600)
run.succeeded, run.output(), run.logs()
```

A job is a different entrypoint of *this same app*, from the same release, in a
container with its own memory limit and deadline. It never blocks on the work.

**Two objects, and which one you call is the decision.** `lp.jobs` is the app
acting as itself. `lp.viewer(request.headers).jobs` is the app acting for the
person looking at the page — attribution that would otherwise be lost by
omission. `viewer(...)` raises when there is nobody to act for.

```python
me = lp.who(request.headers)          # Identity, or None for an anonymous visitor
if me and (me.role == "owner" or me.is_admin):
    render_admin_tab()
if me and "finance" in me.group_names:
    render_approvals()
```

`who` returns `None` for a visitor who signed in nowhere, because that is an
answer. It raises for one thing only: an app an administrator has not switched
viewer identity on for.

`role` is `owner`/`editor`/`viewer` and answers *what may this person do to this
deployment in Launchpad* — not what they may do in your domain. A rule about
invoices belongs on `groups`. `is_admin` is separate on purpose: an
administrator has effective ownership of every app, so `role == "owner"` alone
would grant a destructive action to every administrator on the install. `email`
is `None` on a public app.

### Inside a job, inside a task

```python
lp.run                                # the job's view of itself; None elsewhere
task = lp.task(request.headers)       # a scheduled firing; None on an ordinary visit
if task:
    task.log("read 1,204 rows")
```

A scheduled task is an HTTP request the platform makes to a path you published —
it owns no process, so `task.log(...)` is the only thing that makes that run's
log exist. Worth adding to anything that might outlive the timeout: a run that
never answers is recorded as a failure with nothing else in it.

## Once it is deployed

`lp logs` is what your app printed, `lp events` is what the platform noticed,
and `lp status` says what is serving. A newly created app is private — an
anonymous request to its URL answering `authentication required` is the app
working, not a failed deploy.
