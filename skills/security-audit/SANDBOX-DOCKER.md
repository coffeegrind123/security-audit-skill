# Provisioning the sandbox with sibling Docker containers

Use this when the agent runs inside a container (incl. Docker-in-Docker / WSL2) and its
own namespace calls are denied but a Docker daemon is reachable. It makes the skill's
"Universal execution safety" controls real instead of forcing every lead to
`needs_validation`.

## Diagnose first

```sh
unshare -rn true            # EPERM here => the agent cannot self-sandbox
bwrap --unshare-net --ro-bind / / true   # also EPERM => same
docker version --format '{{.Server.Version}}'   # works => the daemon can sandbox for you
docker run --rm --network none alpine sh -c 'ip addr | grep -c "inet "'  # 1 => loopback only
```

If `unshare`/`bwrap` are EPERM but `docker` works, the container's seccomp/AppArmor
forbids the agent from creating namespaces, yet the **daemon** creates fully-isolated
siblings. Use those; do not conclude "no sandbox."

## Control mapping (each skill requirement → a docker flag)

| Skill requirement | Flag |
|---|---|
| no external network (loopback only) | `--network none` |
| empty allowlisted environment | `--env-file /dev/null` (host env is NOT inherited by `docker run`; add only explicit safe `-e` vars) |
| read-only target + toolchain | `--read-only` (rootfs) + target mounted `-v <repo>:/target:ro` |
| scratch-only writes | `--tmpfs /scratch` (+ `--tmpfs /tmp`); nothing else is writable |
| CPU/mem/proc/file-size limits | `--memory 1g --memory-swap 1g --pids-limit 256 --cpus 2 --ulimit fsize=<bytes>` |
| wall-clock limit | outer `timeout --signal=KILL <sec> docker run …` |
| no privilege escalation | `--cap-drop ALL --security-opt no-new-privileges` |

Reference runner (harness fed via stdin — see gotcha 2):

```sh
timeout --signal=KILL 240 docker run --rm -i \
  --network none --env-file /dev/null --read-only \
  --tmpfs /scratch:size=256m,exec --tmpfs /tmp:size=64m \
  --memory 1g --memory-swap 1g --pids-limit 256 --cpus 2 \
  --ulimit fsize=536870912 --ulimit nofile=1024 \
  --cap-drop ALL --security-opt no-new-privileges \
  -v "$REPO_ON_DAEMON":/target:ro -w /scratch \
  "$IMAGE" sh -s < harness.sh
```

The **toolchain image is built once, with network** (`FROM ubuntu:24.04` + `gcc
libc6-dev make python3`, or reuse a local `postgres:*-alpine` for DB leads). That
one provisioning build is not "the sandboxed check fetching deps" — it builds the
sandbox. Every validation *run* afterward is `--network none`.

## Blockers that cost real time (and the fix)

1. **Bind mount shows empty (9p / Docker Desktop).** A sibling container's `-v`
   resolves on the **Docker host**, not inside the agent's container. On WSL+Docker
   Desktop the repo must be mounted by its Windows path
   (e.g. `//c/users/<user>/…/repo`), not the agent's `/home/...` view (which mounts a
   divergent, often empty tree). Confirm with `ls /target/<a-known-file>` before building.
2. **The harness file is invisible to the daemon.** A script the agent writes to its
   *own* container's `/tmp` (or a session scratchpad) is not on the daemon's
   filesystem, so `-v harness.sh:…` fails the same way as #1. Feed it via **stdin**:
   `docker run -i … sh -s < harness.sh`. (Or write it into the mounted repo tree, in a
   gitignored dir.)
3. **A build/DB engine that refuses root + `--cap-drop ALL`.** `initdb`/`postgres`
   refuse to run as root, and with `--cap-drop ALL` the container cannot `chown` a
   root-owned tmpfs to a service user. Fix: run the container `--user <svc>` (e.g.
   `--user postgres`) and make scratch writable to it with
   `--mount type=tmpfs,dst=/scratch,tmpfs-mode=1777` instead of `--tmpfs`.
4. **Per-connection forks hide the crash.** A server that forks a handler per client
   writes its sanitizer report on the child's stderr. Redirect the whole process
   group's stderr to a scratch file and set `ASAN_OPTIONS=abort_on_error=0:log_path=/scratch/asan`,
   then read `/scratch/asan.*` after the client disconnects.

## Recipes by blocker class

- **Native memory safety (ASAN):** copy the module's sources into `/scratch`, compile
  the *unmodified* sources with `-fsanitize=address -g -O0`, run the real server on
  `127.0.0.1`, drive it with a minimal client, and read the ASAN log. Stop at the
  first out-of-bounds report — do not weaponize it.
- **DB / business logic:** load the repo's schema into a throwaway engine over its unix
  socket (`--user postgres`, `-k /scratch`, `listen_addresses=''`), seed the minimal
  attacker precondition, call the real function, and assert the state change. For logic
  that lives in application code (Node/pg), add the runtime to the image and call the
  real module function, not a reimplementation.

## What the sibling sandbox still cannot settle (legitimate `needs_validation`)

- A **deployment fact** (was a commit pushed to a remote; is a port exposed to
  untrusted peers; is `NODE_ENV`/an allowlist set in prod) — offline evidence only.
- An **external binary/service** not in the repo (its arg parser, its behavior).
- A target that **cannot be compiled** here (e.g. a WASM engine needing an absent
  toolchain) — reason about it from source and hand the maintainer a bounded plan.
