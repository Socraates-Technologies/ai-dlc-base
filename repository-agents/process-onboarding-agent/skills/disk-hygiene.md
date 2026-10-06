# Skill: Disk Hygiene

**Purpose:** Reclaim workstation disk space that the development process consumes passively — stale git worktrees, package-manager caches, build artefacts, simulator images and orphaned container volumes. None of it is reclaimed by any other ceremony: every bolt started on a fresh worktree adds a full dependency install, every container build adds to a cache, and every removed worktree can orphan a volume. Left alone it ends in a full disk, and a full disk fails builds with misleading errors that cost far more time than the sweep.

**Trigger:** Engineer-initiated ("clean up the disk", "what's eating the disk", "run disk hygiene"), or prompted automatically by the AI at session start when the `Next disk hygiene sweep` date in the master rule file Section 9 has been reached. Also run it immediately, unscheduled, whenever free space is observed below the **headroom threshold** in Section 9 (default **25 GiB** — roughly what a fresh worktree plus a full build needs).

**Platform note:** the commands below are written for macOS (BSD userland) with Docker, npm/yarn/pnpm, and the Xcode and Android toolchains. Skip the steps for tools the workstation does not have, and translate paths on Linux or Windows. The rules and the order of the steps carry over unchanged.

---

## The Two Hard Rules

Everything below is reversible except violations of these. Read them before running any command.

1. **Never delete anything holding uncommitted or unmerged work.** Check every worktree with `git status --porcelain` and `git rev-list --count main..<branch>` **before** removing it. If either is non-zero, commit the work to its branch first (Step 2) and remove nothing until it is safe in `.git`.
2. **Never delete a live development database or untracked engineer data.** Two traps:
   - A dev database started with `docker run` **without `-v`** keeps its data on an **anonymous volume** with a 64-hex name, indistinguishable by name from build residue. The only guard is the container cross-check in Step 4. Read the project's `guidelines/dev-setup.md` to learn how its database is started, and run the cross-check every time.
   - Gitignored local data — fixture directories, local database or storage-emulator data folders — **cannot be recovered from `.git`**. Treat it as engineer data, not artefacts.

Corollary: this skill reclaims **caches and artefacts**. Never source, never data, never the engineer's own files. See Step 6.

---

## Step 1 — Measure Before Touching Anything

Record the starting point, so the report at the end is evidence rather than an estimate:

```bash
df -h /System/Volumes/Data | tail -1
```

Then find the real consumers instead of assuming. Work top-down, one level at a time, largest first:

```bash
du -x -d1 -h ~ 2>/dev/null | sort -hr | head -25
du -x -d1 -h /Applications /Library /opt /private/var 2>/dev/null | sort -hr | head -20
find ~ -xdev -type f -size +2G -print0 2>/dev/null | xargs -0 du -h 2>/dev/null | sort -hr | head -20
```

Reading the output:

- **Use `du`, not `ls -lh`, for sparse files.** A VM disk image reports its *apparent* size to `ls`; `du` reports what is allocated.
- **`du -sh` and `-d1` together print nothing on BSD `du`** — the flags contradict each other. Use `-d1` alone.
- **Always pass `-x`.** Without it, `du` descends into mounted volumes and counts a disk image twice — once as the backing file and again through its mount point. In one observed sweep this inflated an estimate from 13 G to 47 G. When a path turns out to be a mount point, take its real size from `df` and its backing file from `du -x`, and reconcile the two before quoting a figure.
- **Present the findings as a table of buckets before deleting anything.** The engineer needs to see when (for example) their own media or asset libraries dwarf every cache combined, so they can make the call on the big items themselves.

---

## Step 2 — Sweep Stale Worktrees

```bash
git worktree list
git branch --merged origin/main
```

Classify each worktree before acting:

| State | Action |
|---|---|
| Clean **and** 0 commits ahead of `main` | Remove: `git worktree remove <path>`, then `git branch -d <branch>` |
| Clean but ahead of `main` | Leave it — it holds unmerged work. Reclaim only its dependency folder (below) |
| Dirty (any uncommitted file) | **Commit to its own branch first**, then re-classify |

Committing a dirty worktree found mid-sweep: stage **explicit paths**, never `git add -A`. When several sessions share one checkout, another session's in-flight work may already be staged in the shared index — follow the project's concurrent-session rule in `rules/code-standards.md`. Use `git -C "$WT"`, never `cd "$WT" &&`: if the directory has been pruned, `cd` fails, the shell stays in the main repo, and the commit lands on the wrong branch.

`git branch -d` (lowercase) refuses to delete an unmerged branch, so it cannot drop real work. **Never use `-D` during a hygiene sweep.**

**Reclaiming the dependency folder without removing the worktree** — the right move for a worktree with unmerged work:

```bash
rm -rf <worktree>/node_modules
```

Know the reinstall cost before doing this. A pnpm reinstall is seconds of hard-linking; an npm reinstall is a real download and build, longer when native modules recompile. Never strip a worktree that a concurrent session is building in.

---

## Step 3 — The Safe Cache Batch

These regenerate on demand and hold no unique state:

```bash
rm -rf ~/.nvm/.cache
rm -rf ~/Library/Caches/Yarn
npm cache clean --force
rm -rf ~/Library/Developer/Xcode/DerivedData/*
```

Others at the same tier, if present: `~/Library/Caches/pnpm`, `~/.expo`, `~/Library/Caches/expo`, `~/.gradle/caches`, `~/Library/Caches/CocoaPods`, `~/.cache/pip`.

Three gotchas:

- **A cache-clean command can refuse to run and still exit 0.** In a pnpm-configured project, Corepack intercepts `yarn cache clean` and prints a message without clearing anything. When a wrapper might intercept, delete the cache directory directly.
- **Under zsh, one glob that matches nothing cancels the whole command line.** zsh's default `nomatch` option makes an unmatched glob an error before the command runs, so `rm -rf <empty dir>/* <cache A> <cache B>` removes nothing: the targets after the glob are skipped with it, and the only trace is a `no matches found:` line that is easy to read past. Keep one `rm` per target, as the block above is written, or `setopt null_glob` before a combined line. bash leaves an unmatched glob literal and is unaffected, which is why this does not reproduce in scripts. _(Ascent, 2026-10-05: an already-empty `DerivedData/*` cancelled a combined line, and the gradle and CocoaPods caches after it survived the batch; the re-measure below caught it.)_
- **Verify by re-measuring, not by reading the command's output.** All of these failure modes are silent, or close to it:

```bash
for d in ~/.nvm/.cache ~/Library/Caches/Yarn ~/.npm ~/Library/Developer/Xcode/DerivedData; do printf '%s\t' "$d"; du -sh "$d" 2>/dev/null | cut -f1 || echo gone; done
```

`~/.nvm/.cache` is the surprise item: it stores Node **source builds**, and a single V8 static library in it can exceed 4 G. It is pure build residue.

---

## Step 4 — Docker

Docker is usually the largest single reclaimable item and the only one in this skill that can destroy data. Start with the inventory:

```bash
docker system df
docker version --format '{{.Server.Version}}'
```

**Order is load-bearing: containers → images → build cache → volumes.** Each tier pins the next. Build-cache layers marked `SHARED` are held by tagged images; images are held by containers; volumes are held by containers. Running the "always safe" tiers first can reclaim **0B** while gigabytes sit behind them — in one observed sweep both safe prunes returned `Total: 0B` in front of 18 GiB, and after the images went, the same build-cache prune reclaimed three times what `docker system df` had reported as the entire cache. **A 0B tier is not evidence that Docker is clean, and `docker system df`'s per-type figures understate what clearing an earlier tier unlocks.**

**Tier 1 — exited containers. Inventory, then remove by name.** An old container's writable layer can dwarf its image (several hundred MB each is common for build containers). Containers are also what keeps images and volumes from being reclaimable:

```bash
docker ps -a --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Size}}'
```

Remove by explicit name. **Never `docker container prune -f`** — it cannot tell a years-old build container from another project's dev database that happens to be stopped, and stopped is the normal state between sessions.

**Tier 2 — images.** Dangling images are safe:

```bash
docker image prune -f
```

**Tagged-but-unused images are not safe by default — ask.** `docker system df`'s "reclaimable" figure counts tagged images with no running container. Two questions decide each one, and only the engineer can answer the second:

- **Is it still pullable?** An image from a registry that no longer exists — a decommissioned cloud account, a former employer's private registry — is gone for good once removed. Check the repository prefix.
- **Does a current project depend on it?** Public base images re-pull in seconds. Anything else is named individually, with its age and size, and needs the engineer's yes.

```bash
docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.CreatedSince}}\t{{.Size}}'
```

**Tier 3 — build cache. Safe, no data risk:**

```bash
docker builder prune -f --filter until=24h
```

The age filter avoids costing a concurrent session a full rebuild for no extra reclaim.

**Tier 4 — volumes. Never run blind.** This is where the data lives. Identify owners first:

```bash
docker ps -a --format '{{.Names}}' | while read c; do v=$(docker inspect "$c" --format '{{range .Mounts}}{{if eq .Type "volume"}}{{.Name}} {{end}}{{end}}'); [ -n "$v" ] && echo "$c -> $v"; done
docker system df -v | sed -n '/Local Volumes space usage/,$p'
```

Read the `LINKS` column (`0` means no container references it):

- **Anonymous volumes (64-hex name, `LINKS 0`)** are old container data directories. Safe to remove **only after the cross-check below** — they can include a dev database's data directory, not just residue.
- **Named volumes with `LINKS 0`** need a human decision; someone named them because they meant to keep them. Present each with its size, and offer to archive it first (below). Never fold them into a bulk command.
- **Volumes with `LINKS ≥ 1`** are in use, and prune will not touch them — but only while the owning container exists.
- **Worktree orphans** are common: Compose derives its project name from the directory, so a removed worktree leaves volumes named `<worktree-dir>_<volume>`. Match them against `git worktree list`. Orphans belonging to **another** project's worktrees are surfaced and left — not this sweep's decision.

On Docker Engine 23.0 and later, `docker volume prune` removes only anonymous volumes by default; `--all` adds named ones. Confirm the engine version rather than assuming, and never pass `--all` without walking the named list with the engineer.

**Prefer an explicit target list over `prune`.** Naming the volumes makes the blast radius provable rather than dependent on engine defaults. Build the list, cross-check it against every container's mounts, and require the intersection to be empty:

```bash
SNAP=<scratch directory>
# candidates: 64-hex name AND zero links
docker system df -v | sed -n '/Local Volumes space usage/,$p' \
  | awk 'NR>2 && NF>=3 && $1 ~ /^[0-9a-f]{64}$/ && $2==0 {print $1}' > "$SNAP/anon-target.txt"
# authoritative in-use set: every volume mounted by ANY container, running or stopped
docker ps -aq | while read c; do docker inspect "$c" --format '{{range .Mounts}}{{if eq .Type "volume"}}{{.Name}}{{"\n"}}{{end}}{{end}}'; done \
  | grep -v '^$' | sort -u > "$SNAP/in-use.txt"
# MUST print nothing — if it prints, stop and re-derive the target list
comm -12 <(sort "$SNAP/anon-target.txt") "$SNAP/in-use.txt"
xargs -n 20 docker volume rm < "$SNAP/anon-target.txt"
```

Two traps in that snippet, both of which produced a **silent** wrong result on first use:

- **`xargs -a <file>` is a GNU extension.** BSD `xargs` rejects it, the removal does nothing, and the surrounding script reports success. Redirect with `< file`, and confirm by re-counting `docker volume ls -q` — never by the absence of an error.
- **A stopped container still holds its volumes.** `docker ps -aq`, not `docker ps -q`, is what makes the cross-check sound. In one observed sweep the in-use set caught four anonymous volumes belonging to dev databases — one running, three stopped. The name-and-links filter alone would have deleted a live dev database.

Then prove nothing load-bearing was lost: every named volume still present, every expected container still up, and the dev database actually answering:

```bash
for v in <named volumes>; do printf '%-42s' "$v"; docker volume inspect "$v" --format 'PRESENT' 2>/dev/null || echo 'MISSING'; done
docker ps --format '{{.Names}}\t{{.Status}}'
docker exec <dev-db-container> pg_isready   # or the project's own health check
```

**Archive-then-delete makes the named-volume decision cheap.** A name and a size cannot tell you whether a volume is a cache or someone's authored work, so look inside. Mount it read-only using an image that is already local, so this costs no pull:

```bash
docker run --rm -v "$v":/v:ro <local-image> sh -c 'du -sh /v; ls -A /v | head -6'
```

Sort what you find into **empty** (nothing to archive), **regenerable** (a tool cache, a devcontainer server install — delete outright) and **data-bearing** (databases, content). Only the third is a real decision, and it is rarely keep-versus-lose, because volume contents compress hard — one observed sweep took 1.3 G of volumes down to 153 M of archives. Offer the archive:

```bash
OUT=<archive directory>/<YYYY-MM-DD>; mkdir -p "$OUT"
docker run --rm -v "$v":/v:ro -v "$OUT":/out <local-image> tar czf "/out/$v.tar.gz" -C /v .
```

**Verify every archive before removing a single source volume.** An unverified tarball is not a backup:

```bash
gzip -t "$OUT/$v.tar.gz"                                     # integrity
tar tzf "$OUT/$v.tar.gz" | wc -l                             # non-empty
tar tzf "$OUT/$v.tar.gz" | grep -E '<the files that matter>' # e.g. base/ and pg_wal/ for Postgres
```

The last check is the one to insist on: confirm the specific files the engineer would grieve are inside, not merely that the archive is well-formed and large. Restore later with `docker volume create <name>`, then `tar xzf` into a mount of it.

**Measure the host, not the VM.** Pruning frees space inside Docker's Linux VM; the host-side disk image may or may not shrink straight away. Check it:

```bash
du -sh ~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw
```

If it has not fallen, reclaim through Docker Desktop's disk settings or a restart. Report the host figure, never just `docker system df`.

---

## Step 5 — The Judgment-Call Tier

Reclaimable, but each item needs the engineer's decision. Present sizes and ask — one item at a time:

| Item | Typical size | Consideration |
|---|---|---|
| `~/Library/Developer/Xcode/iOS DeviceSupport` | 3–6 G per OS version | Regenerates on next device attach. Old OS versions are the obvious first cut |
| `~/Library/Developer/CoreSimulator/Devices` | 10 G+ | `xcrun simctl delete unavailable` is the safe cut. The rest is live device state, so `simctl delete all` needs the engineer to say it explicitly |
| Simulator **runtimes** and their dyld caches | 6–8 G each, plus ~3 G cache each | The largest item in the Xcode stack and the only one with a real re-download cost. See below |
| `~/Library/Developer/Xcode/Archives` | 5 G+ | **Shipped, distributable builds.** Never bulk-delete |
| `~/Library/Android/sdk/system-images`, `~/.android/avd` | 20 G+ | Re-downloadable but slow. Confirm which API levels are still targeted |
| `node_modules` in other projects | 30 G+ in aggregate | All reinstallable. Duplicated trees are the strongest candidates. See below |
| `~/Downloads` | varies | The engineer's call, always |

**The Xcode simulator stack, cheapest to re-acquire first:**

1. **`iOS DeviceSupport`** — free to lose.
2. **Devices** — check `xcrun simctl list devices booted` first and **stop if anything is booted**. Devices are cheap to recreate because runtimes are stored separately. Expect the space to be concentrated in a few devices.
3. **Runtimes** — choose by last-use date:

   ```bash
   xcrun simctl runtime list -v          # Last Used At, Deletable, Size per runtime
   xcrun simctl runtime delete <uuid> [<uuid>…]
   ```

   No sudo and no GUI needed.
4. **`/Library/Developer/CoreSimulator/Caches/dyld`** — do **not** delete by hand. It is root-owned, and `simctl runtime delete` removes each runtime's cache for you.

**Runtime deletion is asynchronous — the first `df` understates it badly.** Right after the command the backing image is unlinked but still mounted, so the space is held open and `simctl` reports `State: Deleting`. In one observed sweep the immediate reading was ~6 GiB and the settled figure ~22 GiB. **Poll `xcrun simctl list runtimes` until the entry disappears, then measure.**

**Sweeping `node_modules` across projects.** Three exclusions are not optional, and all of them are about work in flight:

- **This project's own dependency folders** — they back active development and pre-push hooks.
- **Any live worktree's dependency folder** — a concurrent session may be mid-build.
- **Other active projects on the same workstation** — this sweep cannot see what their sessions are doing.

Build a manifest, exclude by path, and report the excluded set so the engineer can override it:

```bash
find ~/<projects root> -xdev -type d -name node_modules -prune -print 2>/dev/null \
  | grep -v -E '<excluded project paths>' > "$SNAP/nm-target.txt"
tr '\n' '\0' < "$SNAP/nm-target.txt" | xargs -0 du -sc 2>/dev/null | tail -1   # indicative size
tr '\n' '\0' < "$SNAP/nm-target.txt" | xargs -0 rm -rf
find ~/<projects root> -xdev -type d -name node_modules -prune -print | wc -l   # expect only the exclusions
```

`-type d` keeps symlinked `node_modules` out of scope. Afterwards, name the projects that now need an install before their next build.

---

## Step 6 — Never Touch

Do not delete, and do not propose deleting, in any sweep:

- `~/Documents`, `~/Desktop`, `~/Movies`, `~/Pictures`, `~/Music` — source, project data and the engineer's own files.
- Creative asset and sample libraries (audio, 3D, video, design tools). They are often the largest buckets on the volume and are irreplaceable or slow to re-acquire.
- Gitignored local data inside any project (hard rule 2).
- `/Applications` — report the total if it is large, and stop there.
- Any Docker volume backing a database — named **or anonymous** — unless the engineer names it explicitly.

If one of these dominates the volume, **report it and move on.** Archiving it to external storage may reclaim more than every cache combined, but that is the engineer's decision, made with full information.

---

## Step 7 — Report, Then Reschedule

Report the measured delta, itemised, separating what the AI reclaimed from what the engineer reclaimed themselves:

```
Disk hygiene sweep — YYYY-MM-DD

Before:  [N] GiB free ([N] GiB used of [N] GiB)
After:   [N] GiB free
Total reclaimed: [N] GiB   ← the df delta; this is the authoritative figure

Reclaimed (per-item du sizes, indicative):
  [item]                       [N] G
  ...
Verified after:        [named volumes present, containers up, DB reachable, sources intact]
Deferred to engineer:  [items, with sizes and why each needs a decision]
Left untouched:        [user-data buckets, with sizes]
Needs reinstall:       [projects whose dependency folders were removed]
```

**The per-item figures will not sum to the `df` delta.** `du` counts clone- and hard-link-shared blocks at full size, and a disk-image TRIM can land between two measurements. Present the endpoint delta as the number and the items as indicative. If the gap is large, say so and name the cause rather than reconciling it with a guess. And a figure already given to the engineer that turns out wrong is corrected out loud, not quietly restated.

Then ask:

> "When should the next disk hygiene sweep be scheduled? The recommended interval is 30 days. You can give a date or say 'in [N] weeks/months'."

Convert to an absolute date (YYYY-MM-DD); default to 30 days from today if unspecified. Update the master rule file Section 9 rows `Last disk hygiene sweep` and `Next disk hygiene sweep`, adding them if they do not exist:

```markdown
| **Last disk hygiene sweep** | YYYY-MM-DD | Date this sweep was completed |
| **Next disk hygiene sweep** | YYYY-MM-DD | AI prompts at session start on or after this date. Also runs unscheduled when free space is below the headroom threshold |
```

---

## The Principle Behind Every Step

**A command's own output is not evidence. Only a re-measure is.** Every silent failure this skill guards against — a cache clean that no-op'd with exit 0, contradictory `du` flags that printed nothing, a BSD-incompatible `xargs` that removed zero volumes while the script reported success, a prune reading `0B` because another terminal had already pruned — was caught by a count or a size check, never by reading stdout. And every destructive step is followed by asserting the invariant it must not break: volumes present, containers up, database reachable, `package.json` still present in sampled projects. That is what separates a reclaim from an incident.
