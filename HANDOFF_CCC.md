# HANDOFF — SkillsBench intake on CCC (Linux, rootless)

**Status as of 2026-07-29:** plumbing validated end-to-end. A one-task
`bench eval run` smoke completes with `errored=0`, real verifier verdict,
and Claude Opus 4.6 making 7 tool calls. All CCC-specific workarounds are
in place and documented.

**Next action:** run `cap-evolve run --max-iterations 0` (the baseline).

**For a colleague reproducing this on a fresh CCC account:** read
[CCC_PODMAN_SETUP.md](CCC_PODMAN_SETUP.md) — it's a self-contained
walkthrough of every userspace workaround with reasons, a
troubleshooting table keyed to each layer, and a sanity-check script.

---

## For a fresh Claude session picking this up

You are picking up a paused debugging session. Read this whole file first, then
follow "To reproduce from scratch" below to bring the current compute node into
the working state. The blocker is at the aardvark-dns/systemd layer — see
"Where we paused" and "What's still on the table" for the specific next moves.

Key ground truth to know:
- User is `boazc` on IBM CCC (`ccc-loginN` login nodes, `cccxcNNN` compute
  nodes). No admin, no sudo. Can install to `~` and `~/.local`.
- Every host has its own `/tmp` — `podman-561567` graphroot and sockets are
  host-local and rebuilt on each new compute node by sourcing setup_podman.sh.
- `$HOME` and `/dccstor/knewedge2/…` persist across nodes.
- Ask for the current node with `hostname` before making assumptions about
  what's already running.
- `.env` has a real ETE bearer token; treat it as a secret.

You do NOT have SSH access to compute nodes from your login node — ask the
user to paste commands' output when you need it.

---

## The goal (unchanged from parent HANDOFF.md)

Compare `cap-evolve` against **EvoSkills** (arXiv 2604.01687) on
**SkillsBench** (arXiv 2602.12670) with Claude Opus 4.6 + Claude-Code.
Parent HANDOFF was written on macOS; this file records the CCC-specific
port that swapped every "Users/boazc" path for "/dccstor/knewedge2/…"
and worked around every rootless-podman-without-admin blocker.

## Filesystem layout on CCC

| Kind | Path |
|---|---|
| **This worktree** (branch `intake_skillbench_c1`, off `main`) | `/dccstor/knewedge2/boazc/workarea/python/skillberry_ai/cap-evolve-worktrees/intake_skillbench_c1/` |
| Main cap-evolve tree | `/dccstor/knewedge2/boazc/workarea/python/skillberry_ai/cap-evolve/` |
| `.venv` with `cap-evolve` CLI (editable install of ./core) | `.../cap-evolve/.venv/bin/cap-evolve` |
| SkillsBench clone @ pinned commit `9a1f4dd5f7659f75707435da3ce854b6e48321d1` | `/dccstor/knewedge2/boazc/workarea/python/skillberry_ai/cap-evolve-benchmarks/skillsbench/` |
| `bench` CLI (benchflow 0.6.5, from `uv tool install`) | `~/.local/bin/bench` |
| Podman config generator (shared with other projects) | `/dccstor/knewedge2/boazc/workarea/python/setup_podman.sh` |
| Docker Compose v2 (user install) | `~/.docker/cli-plugins/docker-compose` |
| Podman storage/user config | `~/.config/containers/{storage.conf, containers.conf}` |
| Modal auth (unused for now — see below) | `~/.modal.toml` |

## The intake artifacts (all under `.capevolve/project/`)

Fully wired, matching the worked example at `cap-evolve/examples/skillsbench/`:

- `capevolve.yaml`, `capevolve.smoke.yaml`, `split_ids.json`, `smoke_split.json`
- `adapters/adapter.py`, `adapters/anthropic_env.py` (copied from example)
- `optimizer/INSTRUCTIONS.md` (skill-package-scoped optimizer prompt)
- `seed_capability/{docx,pptx,xlsx,pdf}/` (four office skills extracted from the SkillsBench clone; Anthropic-licensed, gitignored)
- `PROJECT.md` (from intake scaffold)

`cap-evolve check .capevolve/project` passes green — 10 val tasks, deterministic scorer, materialize() callable.

## Credentials (in [`.env`](.env), gitignored)

- `ANTHROPIC_BASE_URL=https://ete-litellm.ai-models.vpc-int.res.ibm.com`
- `ANTHROPIC_AUTH_TOKEN=<real; 25 chars, prefix "sk-">`
- `OPENAI_BASE_URL=<same host>`, `OPENAI_API_KEY=<same value>`
- `SKILLSBENCH_AGENT=claude-agent-acp`
- `SKILLSBENCH_MODEL=claude-opus-4-6`
- `SKILLSBENCH_TASKS_DIR=/dccstor/knewedge2/boazc/workarea/python/skillberry_ai/cap-evolve-benchmarks/skillsbench/tasks`  ← CCC path (fixed from Mac's `/Users/boazc/...`)
- `DOCKER_HOST` intentionally NOT hardcoded — sourcing `setup_podman.sh` sets it per node.

---

## Sandbox on CCC: the wall we hit

CCC has no admin, no `/etc/subuid` entry for `boazc`, no systemd user session on compute nodes. Every SkillsBench task builds an OCI image and runs it in a container. Every step of that fought us. Here's the working set of workarounds and where each is applied.

### The full session-setup incantation

```bash
# 1) One-shot per host: source setup_podman.sh — writes storage.conf,
#    starts a private dbus and a podman API socket, exports XDG_RUNTIME_DIR
#    and DOCKER_HOST, and (on first run) builds a patched ubuntu:24.04.
source /dccstor/knewedge2/boazc/workarea/python/setup_podman.sh

# 2) Load creds
cd /dccstor/knewedge2/boazc/workarea/python/skillberry_ai/cap-evolve-worktrees/intake_skillbench_c1
set -a; source ./.env; set +a
```

### The workarounds, layer by layer

Every layer below was needed. Removing any one puts the smoke back to failing.

1. **`/etc/subuid` missing → skip lchown during image unpack.**
   `~/.config/containers/storage.conf` has `[storage.options.overlay] ignore_chown_errors = "true"`. Podman then unpacks `ubuntu:24.04` even though it can't chown `/etc/gshadow` to UID 42.
   Written by `setup_podman.sh`.

2. **`_apt` user (UID 42) can't be setuid to → force apt to stay root.**
   Patched local `docker.io/library/ubuntu:24.04` writes `APT::Sandbox::User "root";` under `/etc/apt/apt.conf.d/00-rootless`.
   Built by `setup_podman.sh` on first run.

3. **Package postinst chowns to weird UIDs (libgd3, fontconfig-config, libc-devtools) → skip Recommends and pre-install what SkillsBench Dockerfiles ask for.**
   Same patched base image also writes `APT::Install-Recommends "false";` and pre-installs `python3 python3-pip curl`. Downstream `RUN apt-get install ...` finds them present and is a fast no-op.
   Built by `setup_podman.sh`.

4. **No systemd user session on compute nodes → private dbus + no-systemd containers.conf.**
   - `setup_podman.sh` spins up `dbus-daemon --session` and exports `DBUS_SESSION_BUS_ADDRESS`. Anaconda's `dbus-daemon` at `~/anaconda3/bin/dbus-daemon` is used.
   - `~/.config/containers/containers.conf` sets `[containers] cgroups = "disabled"`, `[engine] cgroup_manager = "cgroupfs"`, `[engine] events_logger = "file"` — podman avoids dbus/journald paths.

5. **`podman system service` (used by docker-compose v2 to reach podman via API socket) — start idempotently.**
   `setup_podman.sh` starts it in the background if not running, with a pidfile, killing zombies from previous sessions before starting a new one. Multiple zombie services caused a "readonly database" SQLite error earlier.

6. **`docker compose --project-directory ...` v2 syntax → real docker-compose binary as podman's compose provider.**
   `~/.docker/cli-plugins/docker-compose` is Docker Compose v2 (Go binary). `~/.config/containers/containers.conf` sets `compose_providers = [".../docker-compose"]`. Podman-compose 1.5.0 (system-provided) is bypassed.

### Layers resolved (each moved the smoke forward one step)

Full detail with reproduction commands is in
[CCC_PODMAN_SETUP.md](CCC_PODMAN_SETUP.md). One-line summary of each:

1. **Image unpack `lchown /etc/gshadow` → set `ignore_chown_errors = "true"` in `storage.conf`.**
2. **`apt-get install` setuid to `_apt` (UID 42) → patched ubuntu:24.04 with `APT::Sandbox::User "root";`.**
3. **Postinst chowns of libc-devtools/libgd3/fontconfig-config → same patched base adds `APT::Install-Recommends "false";` and pre-installs `python3 python3-pip curl`.**
4. **Dbus missing on compute node → `dbus-daemon --session` started by `setup_podman.sh` + `containers.conf` disables systemd/cgroupfs paths.**
5. **Aardvark-dns needs systemd → base compose yaml patched to `network_mode: host`.**
6. **Readonly SQLite from zombie podman services → `setup_podman.sh` is now idempotent (pkill + pidfile).**
7. **`docker compose cp` chowns to host UIDs that don't exist in container namespace → replaced upload/download with `exec -T` + tar streams (both directions).**
8. **`docker compose exec` treats "agent" user as required → pass `--sandbox-user ''` (root); adapter reads `SKILLSBENCH_SANDBOX_USER` from `.env`.**
9. **`/usr/bin/docker` (podman-docker shim) emits "Emulate Docker CLI using podman..." on stdout, polluting bench's `pwd` probe → userspace `docker` shim at `~/.local/bin/docker` execs podman directly.**
10. **PATH ordering caused the shim to be shadowed** when `~/.local/bin` was already in `$PATH` but after `/usr/bin` — `setup_podman.sh` now explicitly prepends.
11. **Postinst chowns for fontconfig/poppler/cairo etc. through v3 (chown/chgrp wrappers)** — patched base wraps `chown` and `chgrp` to swallow "Invalid argument" errors. Also pre-installs `poppler-utils` and `build-essential` to skip whole classes of downstream apt failures.
12. **Postinst user/group creation for `_dbus`/`messagebus`/etc. (v4)** — patched base also wraps `useradd`/`groupadd`/`usermod`/`groupmod`/`adduser`/`addgroup`.
13. **`dpkg-statoverride` calls `fchown()` via libc during dbus's postinst (v5)** — its "Invalid argument" propagates as a hard dpkg error and cascades through libpam-systemd/gnumeric/libgtk/libgoffice/libreoffice. Wrapped `dpkg-statoverride` too.

### The old "last blocker" (resolved 2026-07-29 evening)

Kept here for context. The list above is the final resolution.

Aardvark-dns is podman's built-in DNS resolver for bridge networks. When the container attaches to a bridge, netavark spawns aardvark-dns via **dbus + systemd** (`org.freedesktop.systemd1` to start a transient scope unit). On compute nodes there is no systemd, so:

```
netavark: error while applying dns entries: IO error: aardvark-dns failed to start:
Failed to start transient scope unit: Process org.freedesktop.systemd1 exited with status 1
```

Container was Created and Starting when this hit. Everything before it (image build, network create, container create) succeeded.

**Fixes tried:**

- Private dbus (fixed the earlier "Failed to connect to bus" error but aardvark still needs systemd on top of dbus).
- `--config-override '{"sandbox":{"network_mode":"no-network","allow_internet":false}}'` — the override was **accepted** (no validation error) but bench's compose-file selection didn't pick it up; the `docker-compose-no-network.yaml` was NOT added to the `-f` list at runtime. Either a bench bug in 0.6.5 or a misunderstanding of which config object drives the compose selection.
- **Edited [`benchflow/sandbox/_compose_files/docker-compose-base.yaml`](/u/boazc/.local/share/uv/tools/benchflow/lib/python3.12/site-packages/benchflow/sandbox/_compose_files/docker-compose-base.yaml) to add `network_mode: none` on the `main` service.** Original is at `docker-compose-base.yaml.orig`. **Even with this, the smoke still errors** — need to check the new error tail (see below).

---

## Where we paused

The last run **as of 2026-07-29 ~09:42 EDT** on compute node `cccxc554`:

- Command exactly as above (`bench eval run --include offer-letter-generator --sandbox docker ...`) with `--config-override '{"sandbox":{"network_mode":"no-network","allow_internet":false}}'` and the site-packages base compose patched to `network_mode: none`.
- Result: still `errors=1`. Full failure text not captured before the session paused. Read the newest artifact dir for the actual error:

```bash
# On cccxc554
D=$(ls -td /tmp/skillsbench-smoke-claude/*/ | head -1)
echo "run: $D"
cat $D/*/results.jsonl | python3 -c 'import sys, json; d=json.loads(sys.stdin.read()); e=d.get("error",{}).get("error_chain_str",""); print(e[-2500:])'
```

Note: `/tmp` is host-local. Artifacts live only on the compute node where the run happened; you can't retrieve them from a login node.

## What's still on the table

If the base-yaml patch didn't fix it (see above), likely next steps:

1. **Check what actually failed with `network_mode: none` in place.** The compose args should NOT include `_default` network creation anymore. If they do, the yaml patch isn't being picked up (maybe cached compose config, or the file was patched wrong — verify with `head -12 .../docker-compose-base.yaml`). If they don't, the new error is a different beast — read the tail, adjust.

2. **If the patch DID take effect but there's a new error:** aardvark isn't the problem anymore. Whatever fails next is a fresh layer — could be podman's per-container network setup, could be volume mounts on GPFS, could be agent-side ACP. Just read the tail and iterate.

3. **If aardvark still fires despite `network_mode: none`:** compose may still auto-create the `default` network. Options:
   - Remove the `networks:` block entirely from the base yaml.
   - Additionally set `--network` at the podman engine level via `containers.conf`.
   - Patch bench code (`sandbox/docker.py` line 263) to always append `docker-compose-no-network.yaml` — the simplest override.

4. **Modal is auth'd but blocked by a separate benchflow bug.** ACP-agent-on-Modal path in benchflow 0.6.5 calls `sandbox.process.exec()` on Modal's Sandbox which uses `.exec.aio()` (no `.process` attribute — see `benchflow/acp/runtime.py:497-501` and `benchflow/sandbox/process.py`). To use Modal instead of docker, write a `ModalProcess` class. Non-trivial (~200 LOC of ACP transport work).

5. **Docker path from a machine with real Docker or working subuid** would skip every workaround above. If you get access to one, ~all the CCC-only files (setup_podman.sh additions, containers.conf, storage.conf, patched-ubuntu base, the base-yaml edit) can be reverted — the smoke would Just Work.

## Files created / modified

Everything below is fully documented in-place with comments about *why*.

- `~/.config/containers/storage.conf` — regenerated by `setup_podman.sh`, now includes `ignore_chown_errors = "true"`.
- `~/.config/containers/containers.conf` — created by hand; disables systemd/dbus paths and points to docker-compose v2 as compose provider.
- `~/.docker/cli-plugins/docker-compose` — Docker Compose v2 binary (~60 MB, from GitHub releases).
- `~/.modal.toml` — Modal auth token (present but unused for now).
- `~/anaconda3/bin/dbus-daemon` — was already there; used by setup_podman.sh.
- `/dccstor/knewedge2/boazc/workarea/python/setup_podman.sh` — heavily extended: private dbus, podman socket, patched-ubuntu base build, idempotent process management. **This is the main knob to rip out if you get admin help.**
- `/u/boazc/.local/share/uv/tools/benchflow/lib/python3.12/site-packages/benchflow/sandbox/_compose_files/docker-compose-base.yaml` — patched with `network_mode: none`. Original saved to `.yaml.orig` beside it.
- `.env` — `SKILLSBENCH_TASKS_DIR` fixed for CCC. Old `DOCKER_HOST` line removed (now set per-node by setup_podman.sh).

## To reproduce from scratch (if disconnected/recovered)

```bash
# All CCC-specific config, one shot
source /dccstor/knewedge2/boazc/workarea/python/setup_podman.sh
# Confirm — none of these should fail:
docker run --rm hello-world
podman run --rm ubuntu:24.04 cat /etc/apt/apt.conf.d/00-rootless
# Expected: APT::Sandbox::User "root"; then APT::Install-Recommends "false";

# Then:
cd /dccstor/knewedge2/boazc/workarea/python/skillberry_ai/cap-evolve-worktrees/intake_skillbench_c1
set -a; source ./.env; set +a
rm -rf /tmp/skillsbench-smoke-claude
bench eval run \
  --tasks-dir "$SKILLSBENCH_TASKS_DIR" \
  --include offer-letter-generator \
  --agent claude-agent-acp --model claude-opus-4-6 \
  --sandbox docker \
  --skill-mode with-skill \
  --skills-dir "$PWD/.capevolve/project/seed_capability" \
  --jobs-dir /tmp/skillsbench-smoke-claude \
  --agent-env "ANTHROPIC_BASE_URL=${ANTHROPIC_BASE_URL:?}" \
  --agent-env "ANTHROPIC_AUTH_TOKEN=${ANTHROPIC_AUTH_TOKEN:?}"
```

## Error log archive (for reference)

Kept in-tree for pattern-matching future failures:

- `errors_1.txt` — image unpack subuid failure (`/etc/gshadow`). Fixed by `ignore_chown_errors`.
- `errors_2.txt` — `apt-get install` failure (setuid to `_apt`). Fixed by patched ubuntu base with `APT::Sandbox::User "root"`.
- `errors_3.txt` — package postinst chowns; `libc-devtools`/`libgd3` failed. Fixed by `--no-install-recommends` + pre-install in patched base.
- `errors_4.txt` — aardvark-dns "Failed to connect to bus". Fixed by private dbus.
- `errors_5.txt` — aardvark-dns "Failed to start transient scope unit: systemd1". **Where we paused.** Attempting to bypass via `--config-override` + `network_mode: none` in base yaml.

## Git state

- Worktree: `/dccstor/knewedge2/boazc/workarea/python/skillberry_ai/cap-evolve-worktrees/intake_skillbench_c1` on branch `intake_skillbench_c1` (off `main` @ commit `020cae2`).
- Everything under `.capevolve/` is gitignored (per repo `.gitignore`).
- `.env` is gitignored.
- Nothing has been committed to `intake_skillbench_c1` branch yet — the intake artifacts are all inside `.capevolve/`.

## Contact points

- Parent HANDOFF (macOS): `/dccstor/knewedge2/boazc/workarea/python/skillberry_ai/cap-evolve/HANDOFF.md`. Read for the "why" behind design decisions (10-task split, 3 trials, optimizer at Opus 4.6, etc.) — those are unchanged.
- Worked example (finished intake, macOS): `/dccstor/knewedge2/boazc/workarea/python/skillberry_ai/cap-evolve/examples/skillsbench/`.
- CI's validated smoke harness: `<worktree>/ci/benchmarks/skillsbench/`.

Good luck.
