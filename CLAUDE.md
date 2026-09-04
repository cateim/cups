# CLAUDE.md

Single source of truth for this repository. Read this before changing anything.

## Overview

`cateim/cups` publishes a multi-architecture Docker image of
[CUPS](https://github.com/OpenPrinting/cups), built from the distro packages of
two bases: Ubuntu Rolling and Debian 13 (Trixie). It is a print server meant to
be dropped into a container host and reached over the network, typically with a
USB printer attached to the host.

There is no application source code here. The deliverable is the image, and the
real product of this repository is the **build pipeline** that keeps it patched
and describes what changed in each rebuild.

- GitHub: <https://github.com/CaTeIM/cups>
- Docker Hub: <https://hub.docker.com/r/cateim/cups>
- Default branch: `master` (never `main`)

## Stack and bases

| Variant | Base image | Notes |
| :--- | :--- | :--- |
| ubuntu | `ubuntu:rolling` | the `latest` track; currently Ubuntu 26.04 (resolute) |
| debian | `debian:trixie-slim` | Debian 13; CUPS version is frozen by the stable release |

Both Dockerfiles install CUPS with `apt-get` and run `apt-get upgrade -y`, so the
image content comes from the **live archive at build time**, not from the base
image snapshot. This single fact drives the whole pipeline design; see
[Build pipeline](#build-pipeline).

## Repository layout

```
.github/
  workflows/build.yml     the entire pipeline (5 jobs)
  builds/                 permanent per-build reports, written by the bot
    <tag>.md              what changed in that build, with changelogs and CVEs
    <tag>.tsv             full package inventory (package/version/source/source-version)
    last-check.json       result of the most recent verification, published or not
  pinned-tags.txt         optional; immutable tags the cleanup must never delete
Dockerfile.ubuntu         ubuntu variant
Dockerfile.debian         debian variant
docker-entrypoint.sh      sets the admin password, seeds /etc/cups, execs cupsd
docker-compose.yml        the manifest Portainer applies, not an example
BUILDS.md                 one line per publication, newest first
CHANGELOG.md              repository changelog (Keep a Changelog 1.1.0)
```

## Build pipeline

`.github/workflows/build.yml`, five jobs. A second workflow,
`.github/workflows/dockerhub-description.yml`, publishes README.md as the Docker
Hub repository description whenever it changes; nothing else keeps that page in
sync, and it is where a first-time `docker pull` gets copied from.

1. **`prepare`** resolves each base to an index **digest** (so amd64 and arm64
   build from the same snapshot), reads the CUPS version from that exact base,
   translates the digest into the official dated tag for readability
   (`ubuntu:resolute-20260811.1` instead of `ubuntu:rolling`), and mints the
   immutable tag name. It does **not** decide whether to build.
2. **`build`** runs one job per (variant x architecture) on a native runner
   (`ubuntu-latest`, `ubuntu-24.04-arm`), so there is no QEMU. It pushes **by
   digest** with no tag, and stamps the OCI labels.
3. **`merge`** reads the package inventory of the new image and of the currently
   published one, diffs them, and **this diff is the publication gate**. If
   something changed, it creates the multi-arch manifest with all tags and the
   index annotations. If nothing changed, nothing is published.
4. **`record`** commits the report to `.github/builds/` and prepends a line to
   `BUILDS.md`. This is also what keeps the schedule alive.
5. **`cleanup`** applies the tag retention policy. Ships with `DRY_RUN: 'true'`.

### Why "rebuild always, publish on diff"

The pipeline used to ask *"does the tag `<cups-version>-<os>` already exist?"* and
skip the build when it did. Because the CUPS version almost never changes
(2.4.16 on Ubuntu for months; Debian stable will not move at all), the answer was
always yes and **the image was frozen from 2026-05-10 to 2026-09** while the
bases were republished twice and the archive received security patches the whole
time. Fifteen consecutive scheduled runs reported success and published nothing.

A gate based on the base image digest would still be wrong, for the reason in
[Stack and bases](#stack-and-bases): `apt-get upgrade` pulls from the live
archive, so patches land between base republications and the digest would not
move. The only honest signal is the real package inventory, and it is only
knowable after the build. Hence: build every week, compare, publish on change.

Consequence: the build runs with `no-cache: true` and there is no `cache-from` /
`cache-to`. With cache, an unchanged base digest returns the old layers, the
`apt-get upgrade` never runs, and the diff is empty forever. Cost is zero because
the repository is public and both runner types are free.

## Tagging policy

Two classes, and the difference is the whole point.

**Immutable**, cut once and never rewritten:

```
<cups-version>-<variant>-<YYYYMMDD>.<run_number>[r<run_attempt>]
2.4.16-ubuntu-20260903.31
```

`run_number` is monotonic and unique per repository, so two builds on the same
UTC day cannot collide. `rN` only appears on a re-run, so a re-run never
overwrites a tag that claims to be immutable.

**Moving**, repointed on every publication: `latest`, `ubuntu`, `debian`,
`<cups-version>-<variant>`.

Every tag is applied by a **single** `docker buildx imagetools create` call. That
is the invariant that makes rollback safe: a moving tag never lands, not even for
an instant, on a digest that has no immutable name.

Rolling back a bad build means **repointing**, never deleting. It writes to the
registry, so it needs a push-capable login first, which a print server host does
not have by default:

```sh
# Without this the command fails with:
#   ERROR: publish ...: failed commit on ref "index-sha256:...":
#   server message: insufficient_scope: authorization failed
printf '%s' "$DOCKERHUB_TOKEN" | docker login -u cateim --password-stdin

docker buildx imagetools create -t cateim/cups:latest -t cateim/cups:2.4.16-ubuntu \
  cateim/cups:2.4.16-ubuntu-<good-date>.<run>
```

Use a token with write scope, the same one CI uses. If you would rather not put a
push token on the server, the alternative needs no login at all: pin the known
good immutable tag (or its digest) in the Portainer stack and redeploy. That
fixes what runs on your host without touching what everyone else pulls, which is
usually what you actually want during an incident.

Retention (`cleanup` job) is an **allow list**, never a block list, so a broken
regex means "deleted nothing" instead of "deleted latest". A tag survives if it
is younger than `KEEP_DAYS` (180), or is among the `KEEP_MIN` (12) newest of its
variant, or is the newest of its CUPS series, or its digest is still pointed at
by any moving tag, or it is listed in `.github/pinned-tags.txt`. A plan larger
than `MAX_DELETE` (10) aborts without deleting anything.

## Patch provenance: how to find out what changed

Five independent layers, on purpose, with different lifetimes. Any one of them
answers the question on its own.

```sh
# 1. index annotations, no download at all, the fastest read
docker buildx imagetools inspect cateim/cups:latest --raw | jq .annotations

# 2. config labels, offline, once the image is pulled
docker inspect cateim/cups:latest --format '{{json .Config.Labels}}' | jq

# 3. the changelog that already ships inside the image
docker run --rm --entrypoint sh cateim/cups:ubuntu \
  -c 'zcat /usr/share/doc/libssl3t64/changelog.Debian.gz | sed "/^ -- /q"'

# 4. the permanent record in git
grep -l 'openssl' .github/builds/*.tsv
grep -rhoE 'CVE-[0-9]{4}-[0-9]+' .github/builds/*.md | sort -u
diff <(cut -f1,2 .github/builds/<older>.tsv) <(cut -f1,2 .github/builds/<newer>.tsv)

# 5. the run's Step Summary, richest and most perishable (90 days)
```

The inventory is read from `/var/lib/dpkg/status`, which every Debian and Ubuntu
image already carries. Nothing is baked into the Dockerfile for this, which is
why the diff also works against images published before this pipeline existed.
Nothing from the published image is ever **executed** in the runner: `docker
create` plus `docker cp` reads the filesystem through the daemon, so a foreign
image never runs next to the Docker Hub token, and it works cross-architecture
without QEMU.

CVE attribution is a `grep` over the changelog text the package maintainer wrote.
It is indicative, never a compliance artifact. Debian stable underreports, since
many fixes ship in point releases with no DSA.

## Triggers and the 60-day rule

- `schedule`: Sundays at 03:00 UTC.
- `push` on `master` only, limited to `Dockerfile.*`, `docker-entrypoint.sh` and
  `.github/workflows/**`. The `branches:` key is **not optional**: without it,
  a push on any branch touching a Dockerfile republishes `latest`, and production
  pulls `latest`.
- `workflow_dispatch`, which always forces publication.

**GitHub disables scheduled workflows after 60 days of repository inactivity.**
Three facts govern the design:

1. The counter measures **commits**. Workflow runs, `workflow_dispatch` and
   `repository_dispatch` do not reset it.
2. When GitHub disables the workflow, it kills **every** trigger in the file,
   `push` and `workflow_dispatch` included. Nothing inside the repository can
   heal it afterwards. Recovery is `gh workflow enable build.yml -R cateim/cups`,
   run from outside.
3. `gautamkrishnar/keepalive-workflow` is **blocked by GitHub for a terms of
   service violation**. What was punished is automated commits whose only purpose
   is to fool the counter.

So the `record` job commits a real audit record, and there is **no empty commit
anywhere in this pipeline**. When a build publishes, the report is the activity.
When nothing changed and the last commit is older than `HEARTBEAT_DAYS` (30),
`last-check.json` is committed, which states a verifiable fact: on this date the
base was X, 334 packages, nothing changed.

Three independent guards stop the bot commit from re-triggering the workflow: the
`paths` filter does not cover `.github/builds/` or `BUILDS.md`; an event made
with `GITHUB_TOKEN` does not start a new run; and the message carries `[skip ci]`.

## Secrets and variables

Names only. Never read, echo or commit a value.

| Name | Where | What for |
| :--- | :--- | :--- |
| `DOCKERHUB_USERNAME` | repo secret | login and the tag delete API |
| `DOCKERHUB_TOKEN` | repo secret | same; needs delete scope for `cleanup` |
| `DOCKERHUB_DESCRIPTION_TOKEN` | repo secret | optional; PAT with `Read, Write, Delete` scope, used only to sync the README to the Docker Hub description. That endpoint rejects a push token with `Forbidden`, so it cannot reuse `DOCKERHUB_TOKEN`. Falls back to it, and fails loudly, when unset. |
| `ADMIN_PASSWORD` | Portainer stack env | password of the container's `admin` user |
| `TZ` | Portainer stack env | container timezone, defaults to `America/Sao_Paulo` |

`.env.example` is the committed contract. `.env` is git-ignored.

## Local development

Windows here is IDE only. Never run `docker build`, `docker compose up` or `act`
locally; there is no container runtime on this machine by design. What can and
should be run before pushing:

```sh
python -c "import yaml;yaml.safe_load(open('.github/workflows/build.yml',encoding='utf-8'))"
actionlint .github/workflows/build.yml      # if installed
bash -n docker-entrypoint.sh
```

Everything that needs a running daemon is validated on the server after deploy.

## Deploy

GitOps. Push to `master`, a Portainer stack using the Repository method pulls the
repo, and a webhook redeploys. There is never a manual deploy step.

Portainer's "Repository reference" field stays at the default `refs/heads/main`
and only lets you pick the branch **after** git authentication succeeds. That is
a UI limitation, not a misconfiguration: it picks up `master` once auth works. If
a stack fails, look at auth, registry or compose, never at that field.

Pointing the stack at `cateim/cups:latest` gives automatic redeploy on every
published patch, which is the intended mode now that `latest` only moves when
something actually changed. Pointing it at an immutable tag trades that for
manual review before each upgrade.

## Container runtime

`docker-entrypoint.sh` sets the `admin` password from `ADMIN_PASSWORD`, seeds
`/etc/cups` from `/etc/cups-skel` when the mounted volume is empty, then
`exec "$@"`, so `cupsd` is PID 1 and receives SIGTERM directly.

The compose file still runs the container `privileged`, which is far more than
printing needs: of the seven things it grants, only unrestricted device cgroup
access is consumed. The image no longer ships `sudo` nor a `NOPASSWD:ALL` sudoers
rule, so the web UI password is no longer a path to root on the host.

The `device_cgroup_rules: ['c 189:* rmw']` recipe that replaces `privileged` is
documented in both READMEs and sits commented in `docker-compose.yml`. It is not
the default because it has to be validated against real hardware, with the
printer power cycled once, and this repository publishes a public image that
other people run on hardware nobody here can test.

Measured on the maintainer's Orange Pi 5: udev gives the printer node `root:lp`
mode `0664`, CUPS backends run as `lp` (uid/gid 7, statically allocated by
`base-passwd`, so it matches inside and outside the container), and the backend
therefore opens the device read-write through the **group** bits. No
`CAP_DAC_OVERRIDE` and no `privileged` are involved in that path.

## Conventions

- Branch is always `master`.
- Conventional Commits, in English.
- **No em dash or en dash anywhere**, in any language, in any file. Use a plain
  hyphen. Sweep before committing:
  `python -c "import re,glob,os;..."` (the `grep -P '[\x{2014}]'` form silently
  returns nothing in Git Bash on this machine and gives a false clean).
- Workflow comments are in Portuguese without accents, matching the existing file.
- `CHANGELOG.md` follows Keep a Changelog 1.1.0 and is written as work happens,
  never generated from `git log`.
- Never commit or push automatically. Leave changes in the working tree.

## Gotchas

Each of these cost an investigation. Do not rediscover them.

- **CRLF kills the container** with `/bin/bash: -e$'\r': invalid option`, not with
  "bad interpreter". `.gitattributes` pins `* text=auto eol=lf` and `*.sh eol=lf`.
- **`git add --renormalize .` alone does not fix the working tree.** With the
  index already in LF it changes nothing. The sequence that works, on a clean
  tree: `git add --renormalize . && git rm --cached -r -q . && git reset --hard`,
  then verify with `git ls-files --eol`.
- **`core.filemode` is false on Windows.** `docker-entrypoint.sh` is `100644` in
  the index and only works because the Dockerfile chmods it. Use
  `git update-index --chmod=+x` to fix it properly.
- **`oci-mediatypes=true` is mandatory** in the build `outputs`. With Docker media
  types, `imagetools create` discards every index annotation silently, no error
  and no warning (docker/buildx#3061). Guard 1 in the merge job catches it.
- **`imagetools create --annotation` only accepts `index:` and
  `manifest-descriptor:`.** A bare annotation or `manifest:` is an error. In the
  build job it is the opposite: use `manifest:`, never `index:`, because a
  push-by-digest build produces no index.
- **Index annotations do not survive the merge**, so they must be applied at
  `imagetools create` time. Labels do survive, because they live in the config
  blob which is copied by digest.
- **`crane tag` preserves the digest, `imagetools create` may not.** Use `crane`
  when giving an existing digest a new name.
- **Never delete an immutable tag by hand** in the Docker Hub UI. Disable a bad
  build by repointing the moving tags.
- **Rollback needs `docker login` first.** `imagetools create` writes to the
  registry, and a print server host has no push credentials, so it fails with
  `insufficient_scope: authorization failed`. Pinning the good tag in the
  Portainer stack is the login-free alternative, and it is usually the better
  one: it fixes your host without changing what everyone else pulls.
- **The workflow id changes on rename.** It already changed once here, from
  `cups.yml` to `build.yml`. Always use the file name in `gh workflow enable`.
- **`declare -A` plus `set -e`**: the `prepare` step deliberately runs with
  `set -uo pipefail` and no `-e`, and the diff step has `-e` off because it uses
  a failing `docker cp` to detect a missing file.
- **`IFS='|'`, never `IFS=$'\t'`** when reading the generated records: tab is
  whitespace to bash's IFS and collapses empty fields, shifting every column.

## Pre-push checklist

1. `python -c "import yaml;yaml.safe_load(open('.github/workflows/build.yml',encoding='utf-8'))"`
2. `bash -n` over every `run:` block that changed.
3. Em dash sweep returns nothing.
4. `git check-ignore -v .env` still matches, and `git ls-files -i -c --exclude-standard` is empty.
5. `CHANGELOG.md` has the entry under `[Unreleased]`.
6. Ask before committing. Always.
