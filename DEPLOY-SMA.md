# Deploy — filestash tester-env

Source-built, single-container environment for `mickael-kerjean/filestash`, used by
the RL training corpus to run nominal UI scenarios with pass/fail mutations.

## Baseline and provenance

| Part | Ref | Notes |
| --- | --- | --- |
| App | `mickael-kerjean/filestash` master `b6fe63f4f01eecbada100d567a84c2503de8fbee` | Pinned by SHA. Upstream publishes **no GitHub Releases** and its only tags (`v0.0`–`v0.4`) end in 2019, so no release tag is usable (see `release_rationale` in `status/apps/filestash.toml`). |
| Fork | <https://github.com/sma-rl-dev/filestash> branch `tester-env-baseline` | Baseline commit above plus one app-only tooling commit. The corpus consumes it as the `apps/filestash` submodule pinned to that commit; every tester-env file lives there. |

No application source is modified by the tooling commit.

```bash
git clone https://github.com/mickael-kerjean/filestash.git
cd filestash && git checkout b6fe63f4f01eecbada100d567a84c2503de8fbee
# copy Dockerfile.tester-env, .dockerignore, tester-env, seed/, seed.manifest.json, DEPLOY-SMA.md
git checkout -b tester-env-baseline && git add -A && git commit -m "tester-env: baseline tooling"
```

## Quick start

```bash
./tester-env deploy    # build image from this checkout, start, seed the drive
./tester-env seed      # (re)apply the deterministic drive + branded config
./tester-env verify    # read-only assertions (health, admin, backend, listings)
./tester-env reset     # remove container + data volume (image kept)
./tester-env stop      # stop container, preserve the volume
./tester-env status    # running?, port, which statics are served
```

Defaults: host port **18650** → container **8334**, container `tester-env-filestash`,
volume `filestash-data`, image `${IMAGE_TAG:-tester-env-filestash:dev}`.
`--run-id <id>` suffixes container/volume for parallel runs; `--port <port>` moves
the host port; `scripts/rl-env` passes both as flags and exports `IMAGE_TAG`, which
the CLI honours verbatim (`RUN_ID` never enters the image name).
Every path is derived from `SCRIPT_DIR`, so the same commands work from a fork
checkout and from `apps/filestash`.

## Deterministic credentials

| Field | Value |
| --- | --- |
| Site name | `Meridian Analytics` (`document.title` on every page) |
| Admin password | `FilestashAdmin!2026` (admin backoffice `/admin/`) |
| Storage backend | connect page: one tab `Meridian Drive`, type `local`, same password |
| Storage path | `/app/filestash-data/drive` |

`ADMIN_PASSWORD` must carry a **bcrypt hash**, not the plaintext: `config.Initialise()`
stores the env value verbatim while `/admin` and the built-in `local` backend
bcrypt-compare the stored value (and `/healthz` requires a 60-char `auth.admin`).
The CLI passes a fixed cost-10 hash of the pinned password, so there is no setup
wizard and reset stays deterministic. `general.secret_key` and the freshly salted
bcrypt hash are random per state dir — assert behaviour, never config bytes.

The `local` entry in `connections` is required: a direct `POST /api/session` is
refused for a backend type the admin config does not list. `seed` writes it
through the same `/admin/api/config` round-trip the admin screens use.

## Build topology (fork-owned `Dockerfile.tester-env`)

Upstream `docker/Dockerfile` `git clone`s master and installs libvips/heif/av
headers so the CGO vips image plugin compiles — not usable as a source build.
This file takes the project's own CI path (`Jenkinsfile`):

1. `generate` stage: `golang:1.26-trixie` produces the gitignored inputs
   (`mime_generated.go`, and the xterm 3.12.2 assets embedded by
   `plg_handler_console`/`plg_widget_console`; those two download from cdnjs once,
   cached in their layer).
2. `build` stage: `sed -i 's|plg_image_c|plg_image_golang|' server/plugin/index.go`,
   writes `server/pkg/env/constants_generated.go` with the pinned SHA (its generator
   shells out to `git` and slices `BUILD_REF[0:7]`; `-buildvcs=false` and no `.git`
   in the context), then `CGO_ENABLED=0 go build -trimpath -buildvcs=false --tags fts5`.
   `/go/pkg/mod` and `/root/.cache/go-build` are BuildKit cache mounts keyed on
   `go.mod`/`go.sum`, so a Go edit recompiles only changed packages.
3. Runtime `alpine:3.22`, static binary, user `filestash`, `EXPOSE 8334`, one
   volume at `/app/filestash-data` (`FILESTASH_PATH`). ffmpeg/poppler-utils are
   deliberately absent (media/office previews out of scope).

Measured on this host (warm base images): cold build **94 s**, Go-only rebuild
**6.7 s**. `public/` is embedded at compile time, but `DEBUG=true` (the CLI
default) mounts `public/` read-only at `/app/public` and the server serves it from
disk, so a frontend edit needs `./tester-env start`, not a rebuild. Set
`DEBUG=false` to exercise the embedded copy.

State: `FILESTASH_PATH` holds `state/config` (admin hash, secret_key,
connections), `state/db`, `state/search` (sqlite FTS), `state/log`, `cache` plus
the seeded drive. `reset` removes the container and that one volume; the image is
kept because content-addressed tags are shared between runs.

## Seed

`seed/drive` is a 10-file / 8-directory small-consultancy shared drive:

```
Shared-Drive-Guide.md
Brand/{palette.json, logo-mark.png}
Contracts/{harbor-health-retainer.md, kestrel-retail-loyalty-cohort-sow.md,
           archive/2025/harbor-health-retainer-2025.md}
Finance/{2026-q3-invoices.csv, budgets/2026-opex-budget.csv}
Team/{onboarding-notes.txt, headshots/avery-chen.png}
```

`seed` owns that directory: it empties `/app/filestash-data/drive`, recopies the
fixtures, pins every file and directory mtime from `seed/mtimes.txt`
(2025-12-18 … 2026-10-02 UTC, `touch -t` in the container, files before
directories) and fails if a fixture has no mtime entry. It then writes
`general.name` and the single `local` connection. `seed.manifest.json` records the
requirement ids and how `verify` covers each one.

## Verify

`./tester-env verify` is 33 checks: `/healthz`, `/about`, admin login, wrong
password refused, admin config readable, local driver + its password/path login
form, `general.name`, connection label, local-backend login, the files page, the
empty-`general.host` origin contract (no redirect, foreign-Host login works), and
for all 9 seeded directories the exact entry names plus every file's size and
mtime milliseconds from `/api/files/ls`. The name comparison means a folder left
behind by a UI action fails verification.

## Browser origin

Containerised browsers do not see `localhost`. Use the docker bridge gateway:

- host / curl / CLI: `http://localhost:18650/`
- Chromium container: `http://<bridge-gateway>:18650/` (here `http://172.16.0.1:18650/`)

`general.host` is **empty by default** on purpose: `SecureOrigin`
(`server/pkg/middleware/http.go`) rejects any non-`/admin` request whose Host
differs from it, and `public/assets/boot/ctrl_boot_frontoffice.js` redirects the
whole frontoffice to it when the hostname differs. The Chromium container cannot
resolve `host.docker.internal`, while host-side checks use `localhost`, so no
single value is safe. Empty means both checks are skipped and every Host works.
`--origin`/`FILESTASH_ORIGIN` pins it (scheme stripped: `general.host` is a bare
`host[:port]`, a URL there renders as `http://http://…` in `/api/config`) — only
use it when absolute links must be fixed and other hostnames are acceptable to lose.

## Scenario metadata

`app_cli` is `tester-env` (submodule root). Because `scripts/rl-env` copies only
`app_cli` + `deploy.copy_paths` from the tooling ref, scenario metas must list:

```json
"copy_paths": ["tester-env", "Dockerfile.tester-env", ".dockerignore",
               "seed", "seed.manifest.json", "DEPLOY-SMA.md"]
```

`seed` (the whole directory) is mandatory: `seed/drive` holds the fixtures and
`seed/mtimes.txt` sits beside it, not inside.

## Evidence

- Source build 2026-11-21: image `sha256:48e5b23cf89e…` reproduced twice from a
  clean tree; `GET /healthz` `{"status": "pass"}`; 33/33 verify after
  `reset && deploy`.
- Browser 2026-11-21 (browser-checker): `http://172.16.0.1:18650/`, title
  "Meridian Analytics", connect tab "Meridian Drive", the full expanded listing
  (folders, sizes, `Shared-Drive-Guide.md` 345 B, no `Scratch`), md/json/csv/png
  viewers opened, no mismatches.
