# Changelog

All notable changes to this project are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project uses CalVer (`YYYY.M.R`).

> **Note:** this version number describes the **repository**, not the published
> image. Image tags carry the CUPS version and the build date; see the tagging
> policy in [CLAUDE.md](CLAUDE.md). Versions below 2026.9.1 were reconstructed
> from git history after the fact.

## [Unreleased]

### Added

- Immutable image tags, `<cups-version>-<variant>-<YYYYMMDD>.<run_number>`, cut
  once and never rewritten. Moving tags (`latest`, `ubuntu`, `debian`,
  `<version>-<variant>`) are repointed in the same `imagetools create` call, so a
  moving tag never lands on a digest without an immutable name.
- Moving tags `ubuntu` and `debian`, so the Debian track finally has a stable
  name that survives a CUPS version bump.
- Package diff as the publication gate: every scheduled run now rebuilds and
  compares the full package inventory against the published image, publishing
  only when something actually changed.
- Patch provenance in five independent layers: OCI index annotations, OCI config
  labels, the package changelogs shipped inside the image, a permanent report in
  `.github/builds/`, and the run's Step Summary. Includes the CVEs cited by the
  package maintainers.
- `record` job, which commits each build report and prepends a line to
  `BUILDS.md`. It doubles as the keepalive for the weekly schedule, using a real
  audit record rather than an empty commit.
- Publication guards in the `merge` job: the run fails if the published index
  lacks the provenance annotations, if the image lacks the provenance labels, or
  if the manifest does not carry exactly `linux/amd64` and `linux/arm64`.
- `ARG BASE_REF` in both Dockerfiles, so CI pins the base by digest and both
  architectures build from the same snapshot.
- `zz-keep-changelogs` in the Debian image, so `debian:*-slim` keeps the package
  changelogs the provenance report reads.
- `CLAUDE.md` and `AGENTS.md`, `CHANGELOG.md`, `.env.example`, and this changelog.
- The 18 printer drivers the image always claimed to ship: `brlaser`, `c2050`,
  `c2esp`, `cjet`, `dymo`, `escpr`, `fujixerox`, `gutenprint`, `indexbraille`,
  `m2300w`, `min12xxw`, `oki`, `pnm2ppa`, `postscript-hp`, `ptouch`, `pxljr`,
  `sag-gdi` and `splix`. See the Fixed section for why they were missing.
- A "Running without `privileged`" section in both READMEs, with the
  `device_cgroup_rules` recipe, how to verify that the USB node carries
  `group=lp`, and the two traps that make this fail silently.
- `.github/workflows/backfill-tags.yml`, a one-shot workflow that gives the three
  already published digests an immutable name with `crane tag`, so rollback stays
  possible once the moving tags start being repointed. Run it once, then delete it.
- `.github/pinned-tags.txt`, an opt-in list of immutable tags the cleanup must
  never delete, for anything that has to outlive the normal retention window.
- `stop_grace_period` and a `logging` block with rotation in `docker-compose.yml`,
  so `cupsd` gets time to flush `job.cache` and `printers.conf` before SIGKILL and
  Docker's json-file cannot grow without bound during a crash loop.
- `.github/dependabot.yml` for the GitHub Actions ecosystem. Minor and patch
  updates are grouped; majors arrive one at a time on purpose, because the
  publishing path was validated against the current versions.
- The publication decision is now echoed to the job log, not only to the run
  summary, so whoever opens the log sees why a run did or did not publish.
- `.github/workflows/dockerhub-description.yml`, which syncs README.md to the
  Docker Hub repository description. Nothing did that before, so correcting the
  README here never reached the page where a first-time `docker pull` is copied
  from. It refuses to publish a README containing relative links, which resolve
  against hub.docker.com and would render as dead links there.
- An upgrade note in both READMEs about the removal of `sudo`, since this is a
  public image and someone may be running `docker exec -u admin cups sudo ...`.

### Changed

- Tag retention rewritten as an allow list with five protections and a
  `MAX_DELETE` brake, shipping with `DRY_RUN` enabled. The previous rule deleted
  anything matching `^(ubuntu|debian)$`, which would now delete the pipeline's
  own moving tags.
- `.gitignore` rewritten from a single line to the full house standard, including
  the environment block with `!.env.example`.
- `.gitattributes` rewritten. It previously referenced `examples/`, `scripts` and
  `.mailmap`, none of which exist in this repository, and its `.git* export-ignore`
  rule stripped `.github/` from the source archive.
- `.dockerignore` rewritten as an allow list; it was the generic `docker init`
  scaffold describing a .NET or Node project.
- Both READMEs: corrected the Debian base from "Testing" to "13 (Trixie)",
  replaced a tag example that never existed on Docker Hub, documented the two tag
  classes, and rewrote the rebuild promise to match what the pipeline now does.
- `docker-compose.yml` no longer carries a literal password. `ADMIN_PASSWORD` is
  now required from the stack environment and the deploy aborts without it.
- OCI labels are now injected by the workflow, replacing the `org.label-schema.*`
  block deprecated in 2018.
- The compose example in both READMEs was realigned with the repository's own
  `docker-compose.yml`; the three copies had drifted apart.

### Removed

- The `sudo` package and the `admin ALL=(ALL) NOPASSWD:ALL` sudoers rule. With
  both in place, anyone who learned the web UI password at :631 became root
  inside the container, and under `privileged: true` that is root on the host.
  Nothing in the print path uses sudo: `cupsd` runs as PID 1 root and is what
  spawns filters and backends. Web UI login is unaffected, because it is
  authorised by membership in `SystemGroup`, which is `lpadmin root`.
- Build headers that nothing ever compiled against: `python3-dev`,
  `libjpeg-dev`, `zlib1g-dev` and `python3-setuptools`.

### Fixed

- Weekly rebuilds no longer skip silently. The gate asked whether the tag
  `<cups-version>-<os>` already existed, and since the CUPS version rarely
  changes, the answer was always yes: the image was frozen from 2026-05-10 while
  fifteen consecutive scheduled runs reported success and published nothing.
- `on: push` is restricted to `master`. Without it, a push on any branch touching
  a Dockerfile republished `cateim/cups:latest`, which is what production pulls.
- Build cache disabled. With cache, an unchanged base digest returned the old
  layers, `apt-get upgrade` never ran, and a rebuild produced a byte identical
  image, so a genuine patch could be published under a stale layer set.
- `fetch-depth` raised from 2 to 0 in `prepare`. On a shallow clone a push of
  more than two commits made the changed-file detection fail silently, so editing
  a Dockerfile did not force a rebuild.
- `docker-entrypoint.sh` changes now trigger and force a build; the file is
  copied into both images but was absent from the path filter and from the gate.
- Digest artifact retention raised from 1 to 5 days, so re-running `merge` the
  next day no longer fails on an expired artifact.
- Added `timeout-minutes` to all five jobs and a guard that fails the merge when
  one architecture is missing, instead of publishing a single-arch manifest under
  the same tag.
- Concurrency group is now per ref, so a test dispatch no longer blocks the
  weekly cycle.
- **The image shipped almost no printer drivers.** `printer-driver-all` is a
  metapackage with 20 Recommends and zero Depends, and the Dockerfile installs
  with `--no-install-recommends`, so it pulled in nothing at all. The only
  drivers present were `foo2zjs` (listed explicitly) and `hpcups` (a real
  Depends of `hplip`). Anyone who pulled this image for a Brother, Epson,
  Samsung, Canon, Lexmark, Oki or Dymo printer never had a driver for it. The
  drivers are now listed one by one, so a driver leaving the archive breaks the
  build loudly instead of vanishing from the image in silence.

## [2026.7.1] - 2026-07-07

### Changed

- Multi-arch builds run natively on amd64 and arm64 runners, pushing by digest
  and merging the manifests, replacing QEMU emulation.
- Disabled provenance attestation to keep `unknown/unknown` entries out of the
  multi-arch manifest.

> This version never published an image. The gate described above kept every run
> after it a no-op, so the tags on Docker Hub still came from 2026.4.2.

## [2026.4.2] - 2026-04-30

### Added

- Conditional builds, tag handling, and automated cleanup of old Docker Hub tags.

### Fixed

- Status code capture in the `check_tag` helper.

## [2026.4.1] - 2026-04-05

### Added

- Portuguese translation of the README.
- Project compose file.

### Changed

- Switched the container to bridge networking.
- Standardised the image build workflow.

## [2025.11.1] - 2025-11-25

### Added

- USB printer troubleshooting guide in the README.

### Changed

- Docker Compose configuration.

## [2025.10.1] - 2025-10-30

### Added

- Debian Dockerfile and the multi-distro build matrix.
- Python and image libraries in both Dockerfiles.

### Changed

- Docker tag generation in the workflow.

## [2025.9.1] - 2025-09-22

### Added

- Initial CUPS image on Ubuntu, the build workflow, the entrypoint script and the
  README.
- `printer-driver-foo2zjs`.
- `linux/amd64` alongside `linux/arm64`.

### Security

- Rebuilt to pick up fixes for CVE-2024-28219, CVE-2025-47273 and CVE-2024-6345.

[Unreleased]: https://github.com/CaTeIM/cups/compare/v2026.7.1...HEAD
[2026.7.1]: https://github.com/CaTeIM/cups/compare/v2026.4.2...v2026.7.1
[2026.4.2]: https://github.com/CaTeIM/cups/compare/v2026.4.1...v2026.4.2
[2026.4.1]: https://github.com/CaTeIM/cups/compare/v2025.11.1...v2026.4.1
[2025.11.1]: https://github.com/CaTeIM/cups/compare/v2025.10.1...v2025.11.1
[2025.10.1]: https://github.com/CaTeIM/cups/compare/v2025.9.1...v2025.10.1
[2025.9.1]: https://github.com/CaTeIM/cups/releases/tag/v2025.9.1
