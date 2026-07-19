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

Copy the example env file and edit it:

```bash
cp .env.example .env
open .env   # or your preferred text editor
````

Set `PROJECT_DIRECTORY` to an **absolute path** on your machine — this directory gets
mounted inside the container at `/projects_mirror`, and is where input data, run
scratch space, and output will live. Leave `IMAGE_TAG` as `latest` unless you need to
pin to a specific FATES/API version (see `CHANGELOG.md` for what's available).

`.env` is gitignored, so your local path never gets committed.

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

The container reads `LOCAL_UID`/`LOCAL_GID` at startup and remaps its internal user to
match your host account, so files it creates in `PROJECT_DIRECTORY` are owned by you,
not some arbitrary container UID — nothing to configure here, `compose.yaml` already
passes your `id -u`/`id -g` by default.

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

`sample_script/e3sm_fates_test.sh` sets up and runs a single-point FATES test case in
Brazil. Copy it into your project directory so it's visible inside the container, and
run it from there:

```bash
mkdir -p "$PROJECT_DIRECTORY/scripts"
cp sample_script/e3sm_fates_test.sh "$PROJECT_DIRECTORY/scripts/"
podman exec -it --user elm-user personal-elm-fates /bin/bash
cd projects_mirror/scripts
chmod +x e3sm_fates_test.sh
./e3sm_fates_test.sh
````

This builds the case, downloads the default input data automatically (`inputdata/`),
runs the model for one simulated year, and archives output (`archive/`) — all under
your `PROJECT_DIRECTORY`. To run your own site instead of the default, edit the script
and point it at your own forcing data.

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

## Versioning

See `CHANGELOG.md` for which release of this repo pairs with which image tag, and what
FATES/host-model versions that image tag contains.
