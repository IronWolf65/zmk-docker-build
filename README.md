# Local ZMK firmware build in a Docker container

**English** | [Русский](README_ru.md)

> ⚠️ This repository was created with AI assistance.

The build runs in the official ZMK Docker container, and the output is firmware
files in `.uf2` format. You don't need to set any extra flags — everything is
taken from the keyboard configuration files.

## Requirements

- Docker Compose
- `make`
- Python 3 on the host — needed for `scripts/select_image.py` (picking a
  compatible image) and `scripts/build.py`
- Internet access: Docker Hub (image) and GitHub (west dependencies on the first build)

## Quick start

```sh
make build
```

This builds all targets from `build.yaml`. On the first run a fairly large
west workspace is downloaded, so the first build takes noticeably longer than
the following ones.

If `build.yaml` or `config/west.yml` is missing, the build stops before
starting Docker and prints the expected keyboard-config layout.

The finished files appear in `output/`:

- `output/<name>.uf2` — what you flash onto the controller (bootloader, double reset);
- `output/other/<name>.*` — other artifacts (`.hex`, `.bin`, `.elf`, `.map`).

`<name>` comes from `artifact-name` in `build.yaml`, otherwise it's composed
from board and shield.

## Commands

| Command | What it does |
|---|---|
| `make build` | Build all targets from `build.yaml` |
| `make build DEV=1` | Same, but in the dev image |
| `make update` | Update west dependencies (`DEV=1` — in the dev image) |
| `make shell` | Interactive bash shell inside the selected image |
| `make clean` | Clean `output/` (does not touch the workspace) |
| `make` / `make help` | Help on the targets |

## Image selection

By default `scripts/select_image.py` picks an image matching the current ZMK
revision:

1. reads the `zmk` project and its `revision` from `config/west.yml`;
2. if that's empty — takes the ref from `.github/workflows/build.yml`, otherwise `main`;
3. downloads `app/west.yml` of that ZMK revision and extracts the Zephyr version;
4. selects `zmkfirmware/zmk-<build|dev>-arm:<Zephyr major.minor>-branch`.

For example, `main` currently uses Zephyr 4.1 → `zmkfirmware/zmk-build-arm:4.1-branch`;
ZMK `v0.3` (Zephyr 3.5) → tag `3.5-branch`. The short form `0.3` also works.

If you need a custom image, set `ZMK_IMAGE` — auto-detection is skipped:

```sh
make build ZMK_IMAGE=zmkfirmware/zmk-build-arm:4.1-branch
```

## How it works

- `build.yaml` — the list of targets in `include`. The local build honors
  `board`, `shield`, `cmake-args`, `artifact-name`, and `snippet`.
- `config/west.yml` — the manifest of ZMK and additional modules.
- `config/xxx.conf`, `config/xxx.keymap` — configuration and keymap.
- `boards/shields/xxx/` — the shield description.
- `docker-compose.yaml` — the container: the repository, `./zmk-workspace`, and
  `./output` are mounted as-is.
- `scripts/build.py` — initializes/updates the workspace, builds the targets, and
  lays out the artifacts.
- `scripts/select_image.py` — image selection (see above).

`zmk-workspace/` and `output/` are not tracked in git. You can delete the
workspace to download it again with a simple `rm -rf zmk-workspace`.
