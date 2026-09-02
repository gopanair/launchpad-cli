# lp — the Launchpad command-line client

`lp` deploys a directory on your disk to a [Launchpad](https://github.com/gopanair/launchpad-install)
install, watches the build, and reads the logs afterwards.

```bash
curl -fsSL https://raw.githubusercontent.com/gopanair/launchpad-cli/main/install.sh | sh
```

One binary. No dependencies, no daemon, no root — it lands in `/usr/local/bin`
if you can write there and `~/.local/bin` if you cannot, and the script tells
you which.

**Linux and Windows.** No macOS build is published yet; the installer says so
rather than failing at a download.

**This repository holds the binaries, not the source.** Each release carries one
build per platform plus a `SHA256SUMS` the installer checks.

## Getting started

```bash
lp login https://launchpad.your-company.com   # opens a browser, once
cd ~/code/my-app
lp link my-app                                # names the app this directory deploys to
lp deploy                                     # pushes what is on your disk
```

`lp deploy` bundles the working tree — respecting every `.gitignore` in it — and
uploads it. It does not need a git remote, a commit, or even a repository; a
plain folder deploys.

## The commands

| | |
|---|---|
| `lp login [url]` | Sign in to an install. `--status` says who and where, `--forget` drops a stored credential, `--token-stdin` for CI |
| `lp link <app>` | Record which app this directory deploys to. `--local` keeps it out of git |
| `lp deploy [dir]` | Bundle and upload. `--logs` tails the app afterwards, `--json` for scripts |
| `lp status` | What is serving, since when, and from what |
| `lp deps` | What the dependency policy says about the app. If a deploy was refused, this is the packages, the advisories and the versions that fix it |
| `lp logs` | The app's log. `--follow`, or `--deploy <id>` for one build's output |
| `lp rollback [id]` | Back to the previous release. Names it before it acts |
| `lp create [dir]` | Make a new app and deploy this directory into it. Tells you the name it will take and asks first. `--repo <url>` to build from a repository instead |
| `lp redeploy` | Build again from the app's own repository. Sends nothing from your disk |
| `lp restart` / `lp start` | Restart the app, starting it if it is stopped. `--wait` waits until it is up and fails if it never comes up |
| `lp stop` | Stop the app |
| `lp events` | What has gone wrong with this app. `--since 2h`, `--severity`, `--kind` |
| `lp job list\|run\|runs\|logs\|cancel` | Jobs, which run in a container of their own with their own memory limit. `lp job run` waits for the job and fails if it fails. `--param k=v` to pass values |
| `lp task list\|run\|runs\|logs` | Scheduled tasks, which are HTTP requests to a path your app publishes |
| `lp apps` | The apps you can see. `--mine` |
| `lp store ls\|put\|get\|rm` | Files in an app's durable store |
| `lp sdk vendor` | The app SDK this install carries, written into your working directory so your editor, `go build` and your CI can see it |

Every command takes `--help`, and every command takes `--json`.

## Reading the output from a script

`--json` puts the answer on standard output and leaves the running commentary on
standard error, so `lp status --json | jq -r .url` works without thinking about
it. Two things are worth knowing:

- **A failed deploy tells you what to do about it.** The result object carries
  `reason` — a short key like `build_failed`, `start_failed` or
  `dependency_blocked` — the whole failure message, and the detail that goes with
  it: the tail of the build log, or, when the dependency policy refused it, the
  packages and the versions that fix them.
- **A refusal is JSON too.** An expired key, a missing app, an install that did
  not answer: under `--json` each writes one object with the message, the exit
  code and what kind of problem it was. You still get the sentence on standard
  error.

## Platforms

| | |
|---|---|
| Linux | x86_64 and arm64 — the install script above |
| Windows | x86_64 — download `lp_windows_amd64.exe` from [Releases](https://github.com/gopanair/launchpad-cli/releases), rename it `lp.exe`, and put it somewhere on your PATH |
| macOS | not published yet |

### Windows

```powershell
curl.exe -fsSLo lp.exe https://github.com/gopanair/launchpad-cli/releases/latest/download/lp_windows_amd64.exe
```

Windows may warn about a program from an unknown publisher the first time you
run it — the binaries are not code-signed yet.

## Configuration

Everything resolves the same way, most specific first:

> `--flag` → `LAUNCHPAD_*` → `.launchpad.toml` → `launchpad.toml` → `~/.config/launchpad/config.toml` → the default

`LAUNCHPAD_URL` and `LAUNCHPAD_APP` are the two environment variables.

**The credential is never a flag.** `lp login` writes it to
`~/.config/launchpad/credentials.toml` at mode `0600`; CI reads it from
`LAUNCHPAD_TOKEN` instead. That file holds credentials and nothing else — the
settings live beside it in `config.toml`, so a file you are meant to edit and a
file with a secret in it never become one file.

## In CI

Mint a personal API key from your install's keys page, put it in the runner's
secret store, and:

```bash
export LAUNCHPAD_URL=https://launchpad.your-company.com
export LAUNCHPAD_TOKEN=lpu_...
lp deploy --no-watch
```

**Give that key full access, not read-only.** A key's scope is chosen when it is
minted and cannot be widened afterwards — a read-only one is replaced, not
changed. It will fetch status and logs quite happily and then fail every deploy
with exit 3, which in a pipeline log looks like a permissions problem with the
account rather than a choice somebody made on the keys page. Signing in with
`lp login` always creates a full key, so this only bites a key pasted in from
the browser.

Where the runner mounts secrets as files rather than exporting them, use
`LAUNCHPAD_TOKEN_FILE` and give it the path — an environment variable is visible
in `/proc`, inherited by every child process and printed by any diagnostic that
dumps the environment, while a path is not sensitive. Setting both is refused
rather than resolved: a job exporting two tokens does not know which one it is
using.

`lp` detects that nothing can answer a prompt — stdin is not a terminal, or `CI`
is set — and exits rather than hanging on a confirmation nobody can see.

A whole pipeline is the same shape. Each step fails the build if it fails, so
you do not need to check anything yourself:

```bash
lp deploy                 # ship it, and wait for the build
lp job run migrate        # run the migration, and wait for it
lp restart --wait         # pick up the new configuration, and wait for it
```

If a step fails, `lp events` says what the platform noticed.

**Exit codes are a contract:**

| | |
|---|---|
| `0` | It worked |
| `1` | The platform did the thing and it failed — a failed build, a refused bundle |
| `2` | Usage or configuration: no app linked, a bad flag, a prompt that could not be asked, or an app an administrator has locked |
| `3` | Not authenticated, or refused |
| `4` | Could not reach the install |

## For coding agents

This repository also carries two skills — the instructions a coding agent needs
to ship an app here without a person watching:

| | |
|---|---|
| [`skills/launchpad-cli`](skills/launchpad-cli/SKILL.md) | signing in without a browser, the deploy loop, the exit codes, what goes in the bundle, and what `lp` deliberately does not do |
| [`skills/launchpad-sdk`](skills/launchpad-sdk/SKILL.md) | writing the app itself: `PORT`, `HOST` and `BASE_PATH`, how the build detects a framework, `launchpad.toml`, and the app SDK for Python and Node |

For Claude Code, drop them where it looks for skills:

```bash
for s in launchpad-cli launchpad-sdk; do
    mkdir -p ~/.claude/skills/$s
    curl -fsSL -o ~/.claude/skills/$s/SKILL.md \
        https://raw.githubusercontent.com/gopanair/launchpad-cli/main/skills/$s/SKILL.md
done
```

**No local runtime is required to deploy.** The install builds: it resolves and
installs dependencies, picks the language version, and starts the app. An agent
with this binary and a key can create an app, push a directory, read the build
log and roll back without Node, Python or a Go toolchain on the machine it is
working on.

## Verifying a download

```bash
curl -fsSLO https://github.com/gopanair/launchpad-cli/releases/download/v0.1.0/lp_linux_amd64
curl -fsSLO https://github.com/gopanair/launchpad-cli/releases/download/v0.1.0/SHA256SUMS
grep lp_linux_amd64 SHA256SUMS | sha256sum -c
```

The installer does this for you.

## Versions

`lp` and the install it talks to share one version number, and an older client
against a newer install is the normal case rather than a problem: `lp` says so
in one dimmed line, renders anything it does not recognise by name, and carries
on. It is never refused for being old.

## Upgrading

Re-run the installer. It downloads the newest published release, checks it
against `SHA256SUMS`, and renames the new binary over the old one — so it is
also how you upgrade a `lp` that is running.

```bash
curl -fsSL https://raw.githubusercontent.com/gopanair/launchpad-cli/main/install.sh | sh
```

`lp --version` says which one you have. To install a particular release, or to
go back to one, pass the version through to the script:

```bash
curl -fsSL https://raw.githubusercontent.com/gopanair/launchpad-cli/main/install.sh | sh -s -- --version 0.1.0
```

`--version latest` follows the newest release rather than the one the script was
published with; `--dir` puts it somewhere else. Nothing else is kept anywhere —
`~/.config/launchpad` is untouched by an upgrade, so you stay signed in.

On **Windows**, download `lp_windows_amd64.exe` again and replace your `lp.exe`
with it. Nothing else changes.

## Uninstalling

```bash
rm "$(command -v lp)"
rm -rf ~/.config/launchpad
```

Deleting the credential file does **not** revoke the keys — that is the keys
page in your install.
