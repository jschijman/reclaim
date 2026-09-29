# reclaim

**Free up gigabytes without losing anything that matters.**

An interactive terminal browser for the caches, dependencies and build output your toolchains
leave behind — grouped by project, ranked by size, deleted on your terms.

`node_modules`, `.venv`, `target/`, `.gradle/caches`, `~/.cache`, `~/.m2/repository`,
`~/.pub-cache`, `~/go/pkg/mod` — every toolchain hoards, and it adds up to tens of gigabytes
scattered across dozens of projects you may not have touched in a year. `reclaim` finds all of
it in one pass, puts **one row per project** (never one per subfolder), sorts by what actually
hurts, and lets you delete it without ever putting your source code at risk.

```
disk /dev/nvme0n1p2   41.2GB free of 467.9GB   90% used   ->   118.7GB free if all rows cleared
reclaim  77.5GB reclaimable in 63 projects

      SIZE  CATS   DIRS  PROJECT
    9.8GB  C         1  .cache
    6.1GB  C         6  .gradle
    4.3GB  CDB      31  Workspace/acme/mobile-app
    3.9GB  C         1  .pub-cache
    2.7GB  DB       12  Workspace/acme/api
    1.4GB  D         3  Workspace/scratch/prototype
...

CATS:  C cache   D deps (re-downloadable)   B build (reproducible)
last scan: 2026-09-29 13:18 (4m ago)   (--rescan to refresh)
```

It is a single self-contained Bash script. No dependencies beyond GNU coreutils and `find`.

> ### Scope
>
> `reclaim` works on **one directory and everything inside it** — your home directory unless you
> say otherwise. It never walks upwards, never crosses onto another filesystem, and can only ever
> delete something it found below that starting point. It is not a whole-disk cleaner and has no
> way of becoming one.
>
> It also **refuses to run as `root`** and **refuses to scan `/` or a system directory**
> (`/usr`, `/etc`, `/var`, …). Run it as yourself, on your own files.

---

## Install

Drop it straight into `/usr/local/bin`:

```sh
sudo curl -fsSL https://raw.githubusercontent.com/jschijman/reclaim/HEAD/reclaim \
  -o /usr/local/bin/reclaim && sudo chmod +x /usr/local/bin/reclaim
```

Then just run:

```sh
reclaim
```

### Install without root

If you would rather not use `sudo`, put it anywhere on your `PATH`:

```sh
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/jschijman/reclaim/HEAD/reclaim \
  -o ~/.local/bin/reclaim && chmod +x ~/.local/bin/reclaim
```

(make sure `~/.local/bin` is in your `PATH`).

### Inspect before installing

Piping a remote script into a privileged path deserves a look first:

```sh
curl -fsSL https://raw.githubusercontent.com/jschijman/reclaim/HEAD/reclaim | less
```

### Update

Re-run the install command — it overwrites in place.

### Uninstall

```sh
sudo rm /usr/local/bin/reclaim
rm -rf ~/.local/state/reclaim     # cached scan results
```

---

## Requirements

- **bash 4+** (uses associative arrays)
- `du`, `df`, `sort`, `numfmt` — GNU coreutils
- `find` — findutils
- `awk` — gawk
- `tput` — ncurses
- `git` — optional, but recommended: without it the ambiguous `build/` `dist/` `target/`
  directories are detected far more conservatively

All of these ship with a default install of essentially every Linux distribution, so in practice
there is nothing to install. If something is genuinely missing, `reclaim` says so on startup and
prints the exact command for your package manager rather than making you work it out:

```
$ reclaim
reclaim needs these commands, which are not installed:

    awk
    numfmt

Install them with:

    sudo apt install gawk coreutils
```

It recognises `apt`, `dnf`, `yum`, `zypper`, `pacman`, `apk` and `brew`, and falls back to
showing both the Debian and Fedora commands when it cannot tell.

Tested on Linux. It should work on macOS with `coreutils` and a modern `bash` from Homebrew
(the stock macOS `bash` is 3.2 and will not run it).

---

## Usage

```
reclaim                 interactive browser
reclaim --list          print the table and exit
reclaim --rescan        force a fresh scan
reclaim --dry-run       never delete, just report what it would do
reclaim --min 100M      hide projects below a size (default 10M, 0 = all)
reclaim --dir DIR       scan DIR instead of your home directory
reclaim --unsafe        skip the root/system-directory refusals
reclaim --help          show this
```

The first run scans your home directory and caches the result. Later runs start instantly from
that cache; press `r` (or pass `--rescan`) when you want fresh numbers.

To narrow it down to one tree, point `--dir` at it — everything below it is fair game, nothing
above it is ever looked at:

```sh
reclaim --dir ~/Workspace
```

If stdout is not a terminal, `reclaim` prints the table and exits, so it composes fine in a
pipeline:

```sh
reclaim --min 1G | tee disk-report.txt
```

### Keys

| Key | Action |
|---|---|
| `↑` `↓` / `k` `j` | move |
| `PgUp` `PgDn` | page |
| `g` / `G` | top / bottom |
| `Enter` / `→` | open the project: choose **ALL** or an individual subfolder |
| `←` / `Esc` | back to the project list |
| `r` | rescan |
| `q` | quit |

Deleting always asks for an explicit `y` first, and shows exactly which `rm -rf` calls it is
about to run and how much they will free.

---

## What it looks for

Everything is classified into three categories, shown in the `CATS` column:

| | Category | Meaning | Examples |
|---|---|---|---|
| **C** | cache | toolchain caches, rebuilt on demand | `~/.cache`, `~/.m2/repository`, `~/.pub-cache`, `~/.cargo/registry`, `~/go/pkg/mod`, `~/.gradle/caches`, `__pycache__`, `.pytest_cache`, `.mypy_cache`, `.tox`, `.turbo` |
| **D** | deps | dependencies you can re-download | `node_modules`, `.venv`, `venv`, `Pods`, `bower_components` |
| **B** | build | reproducible build output | `.dart_tool`, `.next`, `.nuxt`, `.svelte-kit`, `.astro`, `.terraform`, `.stack-work`, plus `build/` `dist/` `target/` |

Nothing here is source code, and nothing here is unique — every one of these directories can be
recreated by re-running a build or a dependency install. That is the whole rule: if it cannot be
regenerated, `reclaim` does not list it.

## Safety

This tool runs `rm -rf`, so its defensive choices are worth stating explicitly:

- **Never as `root`.** `reclaim` exits if it is started with an effective uid of 0. Deleting the
  wrong directory as your own user costs you a rebuild; as `root` it costs you the machine.
- **Never on `/` or a system directory.** `/`, `/usr`, `/etc`, `/var`, `/opt` and the rest are
  refused outright. They are full of paths that look exactly like build output
  (`/usr/lib/node_modules`, `/var/cache`) but belong to your package manager.
- **Never outside the scan directory.** Every deletion target is re-validated at the moment of
  deletion: it must be an absolute path, strictly below the scan directory, contain no `..`, not
  be a symlink, and still be a real directory. Anything else is skipped and reported as skipped.
- **Generic names are guarded by git.** `build`, `dist` and `target` are ambiguous — plenty of
  projects commit a folder with one of those names. Inside a git repo, such a directory is only
  ever listed if `git check-ignore` says the repo itself declares it ignored. Outside version
  control, only `build` and `target` qualify, and `dist` is never touched.
- **Symlinks are never followed** during scanning or deletion, so a symlinked `node_modules` is
  left alone.
- **`.git`, `.svn` and `.hg` are never descended into**, so version-control metadata is out of
  reach by construction.
- **Config directories are preserved.** `~/.gradle` holds `gradle.properties` alongside its
  caches, so the directory itself is skipped and only its regenerable subdirectories
  (`caches`, `daemon`, `native`, `wrapper/dists`, …) are listed.
- **Explicit confirmation**, every time, with the full list and total shown first.
- **`--dry-run`** does the entire run — scan, browse, confirm — and never removes anything.

The first two refusals can be lifted with `--unsafe`. There are legitimate reasons to want that
(a build agent's home under `/var/lib`, a container running everything as uid 0), which is why
the escape hatch exists — but if you are typing it on your laptop, you almost certainly want
`--dir` instead.

Still, this deletes files. Try `reclaim --dry-run` first.

## How it works

1. **One `find` pass**, starting at the scan directory and only ever descending. It stops going
   deeper as soon as it hits an artifact directory, so nested copies are never double-counted,
   and it collects project markers (`package.json`, `Cargo.toml`, `pubspec.yaml`, …) in the same
   walk.
2. **One `du` call** for every candidate, via `--files0-from=-`. Sizes use
   `--block-size=1`, i.e. real disk usage rather than apparent size — with a few million tiny
   files, 4K block rounding is the difference between "12 GB" and what the filesystem actually
   gives back. `-x` keeps it on one filesystem.
3. **Attribution.** Each artifact directory is charged to the nearest ancestor that looks like a
   project root (`.git`, `package.json`, `go.mod`, `pom.xml`, …). Home-level toolchain caches
   stand on their own row instead.
4. **Aggregate and cache.** Results are written to
   `${XDG_STATE_HOME:-~/.local/state}/reclaim/` so subsequent runs are instant. Deleting
   something updates that index in place — no rescan needed.

Drilling into a row shows its individual artifact directories. When a row is a single large
cache (`~/.cache`), it drills one level further and measures the immediate children on demand,
so you can drop just the one 6 GB offender instead of the whole thing.

## License

MIT — see [LICENSE](LICENSE). Use it, fork it, ship it at work, no strings attached.
