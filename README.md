### Disclaimer: this is a personal project and is not supported by the FATES team

This repo lets you download and run a pre-built container image for [FATES](https://github.com/NGEET/FATES)
running on [E3SM](https://github.com/E3SM-Project/E3SM) as host model, without needing
to build anything from source yourself. The image (`ghcr.io/jamedina09/personal-elm-fates-image`)
was built using the Dockerfile from a companion (private) repository, inspired by
[Repo 1](https://github.com/serbinsh/elm_containers) and [Repo 2](https://github.com/NGEET/fates-tutorial).

I personally use [Podman](https://podman-desktop.io/downloads) to manage containers, but
[Docker](https://www.docker.com/products/docker-desktop/) works identically — every
command below works with `docker` substituted for `podman`. I suggest Podman since it's
rootless by default and considered more secure, but either is fine.

## What's actually in the image, and what running it does

The image already contains a compiled GCC/OpenMPI/HDF5/netCDF toolchain and the
E3SM+FATES *source code* — but not a compiled *model*. Step 5 below
(`e3sm_fates_test.sh`) runs `case.build`, which compiles the E3SM/FATES model itself
from that source, inside the container, the first time you run it. That means step 5
is not just "running a script" — it's a real compile step (several minutes) followed
by an automatic input-data download (can be a GB+ depending on what's already
cached) and then the actual 1-year simulation. Budget time accordingly; don't assume
something's stuck just because it's slow.

## Prerequisites

- Podman Desktop or Docker Desktop installed and running.
- A few GB of free disk space for the image itself, plus however much your input data
  and simulation output need (can grow to tens of GB depending on the run).
- No specific hardware requirements to just run the container — pulling and running a
  pre-built image is lightweight. Actual FATES simulations are as demanding as any
  E3SM/CIME run; more CPU cores and RAM will make longer/larger simulations faster.

## Command line Interface (CLI)

You'll interact with this mostly through the terminal (macOS/Linux) or Command Prompt/
PowerShell (Windows). Useful commands if you're new to this:

- `ls` — list files in the current directory (`ls -a` to include hidden files)
- `cd` — change directory
- `mkdir -p` — create a directory (and any missing parent directories)
- `cp` / `mv` / `rm` — copy / move / remove files
- `cat` — print a file's contents
- `chmod +x <file>` — mark a script as executable

## 1. Clone this repo

```bash
mkdir -p ~/projects
cd ~/projects
git clone git@github.com:jamedina09/elm-fates-runner.git
cd elm-fates-runner
````

(No GitHub/SSH set up? Click the green **Code** button on GitHub → **Download ZIP**,
extract it, and move the folder into `~/projects` instead.)

You'll now have:

```bash
elm-fates-runner
├── compose.yaml
├── .env.example
├── CHANGELOG.md
└── sample_script
    └── e3sm_fates_test.sh
````

## 2. Configure

Copy the example env file, then fill in `LOCAL_UID`/`LOCAL_GID` with your own
account's values automatically (don't leave these at the `.env.example` placeholder
of `1000` unless that actually is your UID/GID — mismatched values mean the
container owns files as the wrong user, causing permission errors later):

```bash
cp .env.example .env
sed -i.bak "s/^LOCAL_UID=.*/LOCAL_UID=$(id -u)/; s/^LOCAL_GID=.*/LOCAL_GID=$(id -g)/" .env && rm .env.bak
open .env   # or your preferred text editor, to set PROJECT_DIRECTORY next
````

Set `PROJECT_DIRECTORY` to an **absolute path** on your machine — this directory gets
mounted inside the container at `/projects_mirror`, and is where input data, run
scratch space, and output will live. Leave `IMAGE_TAG` as `latest` unless you need to
pin to a specific FATES/API version (see `CHANGELOG.md` for what's available).

`.env` is gitignored, so your local values never get committed.

The container reads `LOCAL_UID`/`LOCAL_GID` at startup and remaps its internal user to
match them, so files it creates in `PROJECT_DIRECTORY` are owned by you, not some
container default — this only takes effect when the container **starts**, so if you
edit `LOCAL_UID`/`LOCAL_GID` after already running `podman compose up`, you'll need
`podman compose down` then `up -d` again for it to take effect.

`podman compose` (step 4) reads `.env` automatically — nothing more to do for that.
But the plain shell commands later in this README (step 5) reference
`$PROJECT_DIRECTORY` directly, and your shell does **not** read `.env` on its own.
Load it into your current terminal session before continuing (you'll need to repeat
this in any new terminal tab/window you use for these commands):

```bash
set -a
source .env
set +a
````

If you skip this, `$PROJECT_DIRECTORY` silently expands to nothing and commands like
`mkdir -p "$PROJECT_DIRECTORY/scripts"` end up trying to write to `/scripts` at your
filesystem root instead — which fails with a permissions/read-only-filesystem error.

## 3. Pull the image

```bash
podman pull ghcr.io/jamedina09/personal-elm-fates-image:latest
````

This only needs to be run once (or again later to update).

## 4. Run the container

Make sure Podman Desktop (or Docker Desktop) is running, then from this repo's directory:

```bash
podman compose up -d
````

This reads `LOCAL_UID`/`LOCAL_GID` from `.env` (step 2) and remaps the container's
internal user to match, so files it creates in `PROJECT_DIRECTORY` are owned by you,
not some arbitrary container UID.

Confirm it's actually running before continuing — `podman compose up -d` can exit 0
even if the container immediately crashed on startup:

```bash
podman ps -a --filter name=personal-elm-fates
````

Status should say `Up ...`, not `Exited (...)`. If it exited, check why:

```bash
podman logs personal-elm-fates
````

Open a shell inside it — always pass `--user elm-user`, since `podman exec` bypasses
the container's entrypoint (which is what remaps to your UID/GID and sets up
`elm-user`'s environment) and defaults to root otherwise. Running as root breaks
CIME's machine lookup (`ERROR: No machine docker found`), because the custom
`docker` machine config only lives under `elm-user`'s home, not root's:

```bash
podman exec -it --user elm-user personal-elm-fates /bin/bash
````

Inside the container, `cd projects_mirror` and `ls` — you'll see the same files as in
your `PROJECT_DIRECTORY` on the host.

## 5. Run the sample script

`sample_script/e3sm_fates_test.sh` sets up and runs a single-point FATES test case
(`E3SM_FATES_TEST`, compset `2000_DATM%QIA_ELM%BGC-FATES_SICE_SOCN_SROF_SGLC_SWAV`,
resolution `1x1_brazil`) — see "What's actually in the image" above for what this
step really involves (a real model compile, not just a script run). Copy it into
your project directory so it's visible inside the container, and run it from there:

```bash
mkdir -p "$PROJECT_DIRECTORY/scripts"
cp sample_script/e3sm_fates_test.sh "$PROJECT_DIRECTORY/scripts/"
podman exec -it --user elm-user personal-elm-fates /bin/bash
cd projects_mirror/scripts
chmod +x e3sm_fates_test.sh
./e3sm_fates_test.sh
````

This creates the case, compiles the model, downloads the default input data
automatically (`inputdata/`), runs one simulated year, and archives output
(`archive/`) — all under your `PROJECT_DIRECTORY`. To run your own site instead of
the default, edit the script and point it at your own forcing data.

## 6. Check the output

On your host machine (not inside the container), the aggregated output file will be at:

```bash
$PROJECT_DIRECTORY/archive/E3SM_FATES_TEST/lnd/hist/Aggregated_E3SM_FATES_TEST_Output.nc
````

This is a NetCDF file — open it with [Panoply](https://www.giss.nasa.gov/tools/panoply/download/)
(easiest for beginners), NCO, CDO, or Python/R. Commonly-inspected variables: `FATES_GPP`
(gross primary production) and `FATES_VEGC` (total biomass in live plants).

The sample run is just one simulated year for a quick test — real runs can go much longer.

## Stopping and cleaning up

If you're inside the container, run `exit` first to leave it, then from the host:

```bash
podman compose down
````

Anything created *inside* the container outside of `/projects_mirror` is lost when it
stops — but everything under `/projects_mirror` (i.e. your `PROJECT_DIRECTORY` on the
host) persists, since it's a bind mount, not container storage.

## Troubleshooting

Known issues hit while building this workflow, listed by their exact symptom so
you can search/scan for yours:

**`mkdir: /scripts: Read-only file system`** (or any command silently operating on
`/whatever` instead of your actual project path) — `$PROJECT_DIRECTORY` is empty in
your shell. `.env` is only auto-read by `podman compose`, not by your shell directly.
Fix: `set -a; source .env; set +a` (see step 2).

**`ERROR: No machine docker found`**, followed by `cd: /projects_mirror/scratch/...:
No such file or directory` — you `exec`'d into the container without `--user
elm-user`, so you're root, and the custom CIME machine config only lives under
`elm-user`'s home. Fix: always use
`podman exec -it --user elm-user personal-elm-fates /bin/bash` (see step 4).

**`chmod: changing permissions of '...': Operation not permitted`** — usually means
`LOCAL_UID`/`LOCAL_GID` weren't actually picked up when the container started (e.g.
they were missing from `.env`, or you edited `.env` after the container was already
running). Check with `podman exec personal-elm-fates id elm-user` — if the UID/GID
shown doesn't match your host's `id -u`/`id -g`, fix `.env` (step 2) and restart the
container: `podman compose down && podman compose up -d`.

**`Error: can only create exec sessions on running containers: container state
improper`** — the container isn't running (check with `podman ps -a`, then `podman
logs personal-elm-fates` for why it exited). If the log shows `groupmod: GID '...'
already exists`, that's a bug in older image builds — `podman pull` the latest image
to get the fix, then `podman compose down && podman compose up -d`.

**`Error: writing blob: ... denied: permission_denied: The token provided does not
match expected scopes`** (only relevant if you're building/pushing the image
yourself, not just running it) — your GHCR token is missing `write:packages`; see
the build repo's docs.

## Versioning

See `CHANGELOG.md` for which release of this repo pairs with which image tag, and what
FATES/host-model versions that image tag contains.
