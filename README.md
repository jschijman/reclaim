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
- GNU coreutils: `du`, `df`, `sort`, `numfmt`
- `find`, `awk`, `tput`
- `git` — optional, but recommended: without it the ambiguous `build/` `dist/` `target/`
  directories are detected far more conservatively

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
reclaim --root DIR      scan DIR instead of $HOME
reclaim --help          show this
```

The first run scans your home directory and caches the result. Later runs start instantly from
that cache; press `r` (or pass `--rescan`) when you want fresh numbers.

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

- **Generic names are guarded by git.** `build`, `dist` and `target` are ambiguous — plenty of
  projects commit a folder with one of those names. Inside a git repo, such a directory is only
  ever listed if `git check-ignore` says the repo itself declares it ignored. Outside version
  control, only `build` and `target` qualify, and `dist` is never touched.
- **Nothing outside the scan root.** Every deletion target is re-validated at delete time: it
  must be an absolute path, strictly inside the root, contain no `..`, not be a symlink, and
  still be a real directory. Anything else is skipped and reported as such.
- **Symlinks are never followed** during scanning or deletion, so a symlinked `node_modules` is
  left alone.
- **`.git`, `.svn` and `.hg` are never descended into**, so version-control metadata is out of
  reach by construction.
- **Config directories are preserved.** `~/.gradle` holds `gradle.properties` alongside its
  caches, so the directory itself is skipped and only its regenerable subdirectories
  (`caches`, `daemon`, `native`, `wrapper/dists`, …) are listed.
- **Explicit confirmation**, every time, with the full list and total shown first.
- **`--dry-run`** does the entire run — scan, browse, confirm — and never removes anything.

Still, this deletes files. Try `reclaim --dry-run` first.

## How it works

1. **One `find` pass** over the root. It stops descending as soon as it hits an artifact
   directory, so nested copies are never double-counted, and it collects project markers
   (`package.json`, `Cargo.toml`, `pubspec.yaml`, …) in the same walk.
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
