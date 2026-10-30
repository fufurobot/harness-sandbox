# template

<!-- BEGIN BADGE INJECTION -->

[![github](https://img.shields.io/badge/github-fufurobot%2Fmypython--template-181717?logo=github&logoColor=white)](https://github.com/fufurobot/mypython-template) [![branch](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fapi.github.com%2Frepos%2Ffufurobot%2Fmypython-template&query=%24.default_branch&label=branch&color=181717&logo=github&logoColor=white)](https://github.com/fufurobot/mypython-template) [![release](https://img.shields.io/github/v/release/fufurobot/mypython-template?sort=semver&display_name=tag&logo=github)](https://github.com/fufurobot/mypython-template/releases/latest) [![python](https://img.shields.io/badge/dynamic/regex?url=https%3A%2F%2Fraw.githubusercontent.com%2Ffufurobot%2Fmypython-template%2Fmain%2F.python-version&search=%5E%28.%2B%29%24&label=python&color=3776AB&replace=%241&logo=python&logoColor=white)](https://www.python.org/downloads/) [![managers](https://img.shields.io/badge/dynamic/yaml?url=https%3A%2F%2Fraw.githubusercontent.com%2Ffufurobot%2Fmypython-template%2Fmain%2F.github%2Fworkflows%2Fdevcontainer.yml&query=%24.jobs.scripts.strategy.matrix.manager%5B%2A%5D&label=managers&color=dea584)](https://github.com/fufurobot/mypython-template) [![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/fufurobot/mypython-template/main) [![codespaces](https://img.shields.io/badge/codespaces-open-181717?logo=github&logoColor=white)](https://codespaces.new/fufurobot/mypython-template?quickstart=1) [![ci](https://img.shields.io/github/actions/workflow/status/fufurobot/mypython-template/devcontainer.yml?branch=main&logo=githubactions&logoColor=white)](https://github.com/fufurobot/mypython-template/actions/workflows/devcontainer.yml) [![license](https://img.shields.io/github/license/fufurobot/mypython-template?logo=gnu&color=blue)](https://github.com/fufurobot/mypython-template/blob/main/LICENSE) [![last commit](https://img.shields.io/github/last-commit/fufurobot/mypython-template?branch=main)](https://github.com/fufurobot/mypython-template/commits/main) [![TestPyPI](https://img.shields.io/pypi/pyversions/mypython-template-fufurobot?pypiBaseUrl=https://test.pypi.org&logo=pypi&logoColor=white)](https://test.pypi.org/project/mypython-template-fufurobot/) [![dev dependencies](https://img.shields.io/badge/dev_dependencies-2-blue)](requirements-dev.txt)

<!-- END BADGE INJECTION -->

# harness-sandbox

**Capability-aware process sandboxing for local, remote, and agent-harness execution.**

`harness-sandbox` confines subprocess filesystem effects on whatever host you're actually on — a Linux box, a WSL VM, a Windows workstation, an MSYS2 shell, a Termux phone, or a Slurm compute node reached over SSH — using the strongest confinement primitive that host can actually provide. It ships a first-class integration with [Harbor](https://harborframework.com), the open-source agent-evaluation harness behind Terminal-Bench, SWE-bench, tau3-bench, and OSWorld.

It does **not** branch on distribution names. It probes for kernel features, LSM policy, container context, and tool availability, then picks a backend and reports honestly how strong that backend is.

```python
from harness_sandbox import Sandbox, SandboxPolicy

sandbox = Sandbox()                          # probes the local host on first use
plan = sandbox.plan(
    SandboxPolicy(mode="workspace-write", workspace_root="/home/me/project"),
    min_strength="strong",
)
print(plan.backend, plan.enforcement)        # e.g. "bwrap+landlock full"
result = sandbox.execute(plan, ["python", "build.py"])
```

---

## Why

Agentic and build-tool harnesses need a *reliable* answer to "may this subprocess write here?" across machines that have almost nothing in common:

- a dev laptop with Bubblewrap and a modern Landlock ABI,
- a CI container that blocks `CLONE_NEWUSER`,
- a Windows box with no LSM at all,
- an HPC login node that only has a scheduler,
- a phone with a seccomp filter that silently drops Landlock syscalls.

Most sandbox libraries either target one platform or degrade silently when the primitive is missing. `harness-sandbox` treats **capability detection as the primary artifact** and **refuses to run unconfined** unless you explicitly opt in. Harbor's existing backends (Docker, Daytona, Modal, E2B, Vercel, Novita, LangSmith, GKE) all assume a container or VM runtime; `harness-sandbox` covers the hosts where none of those exist.

---

## Design principles

1. **Probe capabilities, never branch on distro.** `distro.id()` is a diagnostic string, not control flow. What matters is Landlock ABI, userns availability, LSM policy, container context, and which tools exist.
2. **Transport ⊥ backend.** *How* you reach a host (local, SSH, Slurm, PBS/SGE) is orthogonal to *how* you confine on it (bwrap, Landlock, ACL token, seatbelt). The remote host's backend is chosen from the remote host's capabilities.
3. **Fail closed.** No silent unconfined passthrough. If no backend satisfies the policy, raise `SandboxUnavailableError` before spawning anything.
4. **Report enforcement honestly.** Backends return `full | partial | none`. Known gaps (Windows hard links, ambient ACLs, drvfs mounts, proot) are surfaced as `partial`, never normalized to `full`.
5. **Canonicalize before containment.** Every path is resolved to its real kernel identity before any writable-root comparison. Symlink and MSYS2 path-aliasing escapes are bugs, not edge cases.
6. **Never mutate the host's image store without consent.** Images are neither downloaded nor built unless you explicitly request it. See [Container spawner detection](#container-spawner-detection).

---

## Supported runtime families

The family narrows the candidate backend set; **probes select within it**; unknown fails closed.

| Family | Kernel / userland | Candidate backends | Enforcement ceiling |
|---|---|---|---|
| `linux` | Linux + glibc/musl | `bwrap+landlock`, `bwrap`, `landlock`, `podman`, `docker` | `full` |
| `wsl` | WSL2 Linux VM or WSL1 syscall translation, glibc | `bwrap+landlock`, `bwrap`, `landlock`, `podman` (WSL1 probes to none) | `full` on ext4, `partial` on drvfs, `none` on WSL1 |
| `darwin` | XNU + libSystem | `seatbelt` | `full` |
| `windows` | NT + Win32 | `acl-token`, `appcontainer` | `partial` |
| `msys2` | NT + msys-2.0 (also Cygwin) | `acl-token`, `appcontainer` | `partial` |
| `termux-native` | Android Linux + bionic | `landlock` | `partial` |
| `termux-proot` | Android Linux + proot'd glibc/musl | `landlock-in-rootfs` | `none`–`partial` |

Detection order: `msys`/`cygwin` → `win32` → `darwin` → Linux, then `WSL_DISTRO_NAME`/`microsoft` in `/proc/version` → `wsl`, then `$PREFIX` under `/data/data/com.termux` → `termux-native`, else `linux`. `termux-proot` is selected by the **launcher**, not by introspection from inside the chroot.

---

## Backends

Ranked by strength, not availability order.

| Backend | Mechanisms | Needs userns | Typical use |
|---|---|---|---|
| `bwrap+landlock` | mount namespace + LSM allow-list | yes | default on modern Linux |
| `bwrap` | mount namespace only | yes | kernels without Landlock |
| `landlock` | LSM allow-list only | **no** | containers, HPC nodes, Termux |
| `podman` | rootless container boundary | no | fallback when bwrap is unavailable |
| `docker` | container boundary | no | when a daemon is reachable |
| `seatbelt` | `sandbox_init` profile | no | macOS |
| `acl-token` | `WRITE_RESTRICTED` token + ACLs | no | Windows, MSYS2 |
| `appcontainer` | AppContainer profile | no | Windows, MSYS2 |
| `landlock-in-rootfs` | Landlock installed inside a proot rootfs | no | Termux + proot |

`landlock` alone is a first-class backend, not a degraded branch of `bwrap` — it is the fallback exactly where `bwrap` cannot run, and that asymmetry is the core of the design.

---

## Container spawner detection

Some hosts have no usable LSM or userns — but they *do* have a container runtime on `PATH`, or a scheduler that can allocate one. `harness-sandbox` treats container spawners as a separate, opt-in axis of capability detection. The rule is short:

> **We never download or build images without your explicit say-so.**
> **We may use containers that already exist and that we can reach.**

This is a deliberate stance against the common failure mode where a sandbox "helpfully" pulls a base image from a registry, silently consumes gigabytes, and mutates the host's image store — or, on an HPC filesystem, blocks for minutes on a network pull the user never asked for. If you want an image, you tell us. If you don't, we use what's there.

### What detection does

The probe walks `PATH` for known OCI-compatible clients and records them in `Capabilities.container_spawners`, each with an `Access` tri-state (`allowed | denied | unknown`) and a reason:

- **`docker`** — `/var/run/docker.sock` connect + `docker version` (cached).
- **`podman`** (rootful and **rootless**) — `podman info --format json`; rootless additionally checks `newuidmap`/`newgidmap` and `/etc/subuid` + `/etc/subgid` entries, because those are the classic silent-failure points.
- **`singularity`** / **`apptainer`** — `singularity --version` / `apptainer --version`; HPC-rooted, often the only option on a compute node.
- **`kubectl`** (and `oc`) — kubeconfig presence + `kubectl auth can-i create pods`, cached.
- **`nerdctl`**, **`ctr`**, **`crictl`** — best-effort, same socket/endpoint discipline.
- **`nvidia-container-toolkit`** — detected as a *capability modifier*, not a spawner: if present, spawners that support `--gpus` / `--runtime=nvidia` are annotated so the planner can honor a GPU requirement. Absence is a hard signal, presence is not a guarantee.
- **`enroot`**, **`udocker`**, **`sarun`** — detected opportunistically; treated as OCI clients if they answer `--version` consistently.

Additionally, container spawners reachable over **non-local transports** are probed by the *same* mechanism:

- **Passwordless SSH** — the remote probe script runs `command -v docker podman singularity apptainer kubectl …` and the corresponding permission checks on the far end, so a laptop can plan a container spawn on `node01` using *node01's* spawners, not its own.
- **Slurm / PBS / SGE** — the scheduler is itself the spawner. The planner can request a container via `srun`/`sbatch` when the login node's filesystem probe confirms the image (or an existing containerized runtime) is reachable from compute nodes. This is subject to the same **shared-filesystem validation** as any other HPC path — an image that exists on the login node but not on the compute node is `partial`, not `full`.

### How permission is verified

We reuse existing tools on `PATH` — nothing is bundled, nothing is fetched. To confirm that a spawner is actually *usable* by the current user, we run a **minimal, local-only** check against a tiny **hello-world image that is already in the local store**:

```text
<runtime> image inspect <hello-world>   # → present?  then:
<runtime> run --rm <hello-world>        # → exit 0?
```

The probe is deliberately cheap and side-effect-free: it never touches the network, never pulls, never builds.

**If no hello-world image is present locally**, we cannot prove spawn permission without a network pull, so we **stop and ask** rather than deciding for you. The CLI surfaces this as an actionable prompt:

```text
$ python -m harness_sandbox probe --pretty
…
container_spawners:
  docker:    allowed   (socket 0660, daemon up, hello-world run ok)
  podman:    unknown   (daemonless; hello-world image NOT present locally)
             → run `harness-sandbox probe --allow-pull hello-world` to verify,
               or set `HARNESS_SANDBOX_ALLOW_PULL=1` to opt in globally.
  singularity: allowed (hello-world present, exec ok)
  kubectl:   denied    (no kubeconfig context)
```

The corresponding policy knobs are:

| Knob | Default | Meaning |
|---|---|---|
| `allow_pull` | `False` | May **download** an image to verify spawn permission |
| `allow_build` | `False` | May **build** an image (`docker build`, `buildah`, `podman build`) |
| `use_existing` | `True` | May use images / containers **already present** on the host |
| `hello_world_image` | runtime default | Which minimal image to use for the permission probe |

These are per-call, not global — the same way `SandboxPolicy` is resolved per capability call, `ContainerSpawnPolicy` is resolved per `plan()`. A caller that is fine pulling for a `docker` plan can forbid pulling for a `podman` plan in the same process.

### OCI compatibility

We try our best to be compatible with any container management tool that obeys the **OCI runtime/image spec**, and we treat "spawner" as an interface rather than a product name. Currently:

| Category | Tools |
|---|---|
| OCI runtimes | `runc`, `crun`, `youki` (indirectly, via clients) |
| Container engines | `docker`, `podman` (rootful + rootless), `nerdctl`, `containerd` (`ctr`), `cri-o` (`crictl`) |
| HPC-native | `singularity` (2.x and 3.x), `apptainer`, `enroot`, `udocker`, `sarun` |
| Orchestrators | `kubernetes` (`kubectl`, `oc`), `nomad` (best-effort) |
| GPU | `nvidia-container-toolkit`, `--gpus` support annotation |
| Registry-agnostic | anything exposing `image inspect` / `run` / `exec` under OCI semantics |

The planner does **not** require a specific tool; it requires an OCI-compatible command surface. A backend that speaks OCI gets the same `ContainerSpawnPolicy` treatment as Docker. Where a tool diverges from OCI semantics in a way we can detect, the planner marks it `partial` and records the divergence in `warnings` — never silently falls back.

### What this buys you on Harbor

Harbor tasks that ship a `docker-compose.yaml` or a `Dockerfile` are still rejected by `HarnessSandboxEnvironment` up front — a host-native process sandbox cannot run multi-container task fixtures, and pretending otherwise produces trials that fail for reasons unrelated to the agent. But a **single-container** task whose image is *already present* on the host — or *already running* on a reachable remote host — is a legitimate target. The integration is:

```bash
# use the existing docker image; do not pull or build anything
harbor run -d terminal-bench@2.0 \
  --environment harness_sandbox.harbor:HarnessSandboxEnvironment \
  --env-arg min_strength=strong \
  --env-arg use_existing=true \
  --env-arg allow_pull=false \
  --env-arg allow_build=false
```

If a trial needs an image that is not present, the run fails with a typed, actionable error — not a background pull — exactly the same shape as the CLI prompt above.

---

## Install

```bash
pip install harness-sandbox            # core library
pip install "harness-sandbox[harbor]"  # + Harbor integration
```

Runtime deps are deliberately thin: `distro`, `platformdirs`, `filelock`, plus `ctypes` for Landlock and Windows APIs. Harbor is an optional extra.

---

## Quick start

```python
from harness_sandbox import Sandbox, SandboxPolicy

sandbox = Sandbox()

caps = sandbox.capabilities()
print(caps.runtime_family, caps.landlock_abi, caps.userns)
print(caps.container_spawners)               # {'docker': 'allowed', 'podman': 'unknown'}

plan = sandbox.plan(
    SandboxPolicy(
        mode="workspace-write",
        workspace_root="/home/me/project",
        min_strength="strong",        # 'strong' | 'partial' | 'any'
        allow_unconfined=False,       # must be True to accept enforcement='none'
        container_spawn=ContainerSpawnPolicy(
            allow_pull=False,         # never download an image by default
            allow_build=False,        # never build an image by default
            use_existing=True,        # do use what's already on the host
        ),
    )
)
result = sandbox.execute(plan, ["cargo", "build"])
```

Confined argv can also be produced without executing, for callers that spawn the subprocess themselves:

```python
argv = sandbox.confine(plan, ["python", "script.py"])
subprocess.run(argv, check=True)
```

---

## Harbor integration

[Harbor](https://harborframework.com) drives every sandbox through one abstract class, `BaseEnvironment` (`exec`, `upload_file`, `upload_dir`, `download_file`, `download_dir`, `start`, `stop`). `harness-sandbox` ships `HarnessSandboxEnvironment`, a host-native Harbor environment that compiles every `exec` through the capability-aware planner.

This is for hosts Harbor's existing backends cannot reach: HPC login and compute nodes without a container runtime, Windows workstations, MSYS2 shells, and Termux devices. It is **not** a replacement for Harbor's Docker, Daytona, Modal, E2B, or Vercel backends, and it deliberately rejects tasks that require a built image, `docker-compose.yaml`, GPU, or GUI — unless the required image is already present on the host and `use_existing=true` is set.

Enforcement is reported honestly. `HarnessSandboxEnvironment.enforcement` returns `full`, `partial`, or `none` from the selected backend, and a `min_strength` floor fails the trial before the agent starts rather than silently running under a weaker boundary. Timeouts follow Harbor's contract (process-group kill, exit code 124), and file transfer is tar-faithful with all paths canonicalized under the trial workspace.

There are three ways to use it.

### 1. Explicit custom sandbox (recommended)

Harbor resolves custom backends from a `module.path:ClassName` string, so no patching is needed:

```bash
harbor run -d terminal-bench@2.0 \
  --environment harness_sandbox.harbor:HarnessSandboxEnvironment \
  --env-arg min_strength=strong \
  --env-arg allow_pull=false \
  --env-arg allow_build=false
```

### 2. Drop-in replacement

`harness_sandbox.harbor` re-exports the Harbor-compatible surface (`BaseEnvironment`, `ExecResult`, `HarnessSandboxEnvironment`), so code written against the Harbor package can point at it instead:

```python
import harness_sandbox.harbor as harbor

env = harbor.HarnessSandboxEnvironment(
    task_dir="./task",
    min_strength="strong",
    allow_pull=False,
    allow_build=False,
)
```

### 3. Monkey patching

For harnesses you don't control — where Harbor's backend is resolved internally and there is no `--environment` hook — patch the process:

```python
import harness_sandbox.harbor as hs_harbor

handle = hs_harbor.patch_harbor()   # idempotent; returns a handle
# ... run Harbor ...
handle.unpatch()                    # restore the original resolution
```

`patch_harbor()` replaces Harbor's environment resolution so that `harbor run` invocations inside the current Python process use `HarnessSandboxEnvironment`. It is **process-local**, affects only Harbor instances created after the call, and is reversible via `handle.unpatch()`. Setting `HARNESS_SANDBOX_HARBOR=0` in the environment makes `patch_harbor()` a no-op, which is useful in CI.

> A host-native process sandbox cannot run `docker-compose.yaml`, build task images, or provide GPU/GUI passthrough. `HarnessSandboxEnvironment` rejects those tasks up front rather than failing them mid-trial. The correct v1 surface is **host-native, single-process-tree tasks** — shell, Python, Node, compilers already installed — plus everything on HPC and Windows where Docker isn't an option, plus **single-container tasks whose image is already present** when `use_existing=true`.

Concurrency is already handled: Harbor runs each trial in a fresh environment instance, and per-call policy resolution was designed for exactly that — two concurrent confined executions can carry different modes and roots to the same provider without state mutation. Each `HarnessSandboxEnvironment` owns one trial's workspace and one `(session, workspace)` temp root.

---

## CLI

Debugging runtime detection is the main workflow, so the probe is exposed directly:

```bash
python -m harness_sandbox probe                 # JSON capability dump for this host
python -m harness_sandbox probe --pretty
python -m harness_sandbox plan --mode workspace-write --root .
python -m harness_sandbox probe --ssh node01    # remote probe over SSH
python -m harness_sandbox probe --pretty --allow-pull hello-world
python -m harness_sandbox probe --allow-build   # opt in to build-based verification
```

---

## Architecture

```
probe ──▶ Capabilities ──▶ planner ──▶ ExecutionPlan ──▶ backend ──▶ transport ──▶ process
```

**Probe** runs platform-specific checks — the Landlock syscall, a `userns` fork, LSM sysctls, container markers, a Unix-socket connect for Docker, `sbatch` presence, `PATH` walks for OCI clients, hello-world run verification (only when opted in), and a constant versioned remote shell script for non-local transports — and emits a serializable `Capabilities` object, cached by `(transport, host)`.

**Planner** takes `(policy, capabilities, preferences)` and returns an `ExecutionPlan` carrying the backend, its parameters, `enforcement`, and a list of warnings. It never downgrades silently, and it never initiates a network fetch or a build.

**Backend** compiles the plan into an argv (or a shell script for schedulers) that enforces the policy.

**Transport** executes it — locally, over SSH (with `BatchMode=yes` and `ControlPersist`), or as a scheduler job.

Key pieces of `Capabilities`:

```python
@dataclass(frozen=True)
class Capabilities:
    schema: int
    runtime_family: str           # linux | wsl | darwin | windows | msys2 | termux-native | termux-proot
    kernel: KernelInfo
    landlock_abi: int             # 0 = absent
    userns: Access                # allowed | denied | unknown  (+ reason)
    apparmor: Access
    selinux: Access
    virt: str                     # bare | vm | container | wsl
    inside_container: ContainerInfo | None
    container_spawners: dict[str, SpawnerInfo]   # docker | podman | singularity | kubectl | ...
    tools: dict[str, ToolInfo]
    launchers: list[str]          # slurm | pbs | sge | k8s
```

`Access` is tri-state with a reason, because *unknown* (probe timed out, image absent, daemon unreachable) must fail closed differently from *denied*.

Paths are resolved once per family and injected into every backend, so Termux's `$PREFIX/tmp`, macOS's `$TMPDIR`, MSYS2's `/c/… → C:\…` translation, and proot's host-side root are not special-cased inside individual backends:

```python
@dataclass(frozen=True)
class PlatformPaths:
    tmp_root: Path
    shell: Path
    libc: str | None              # "libc.so.6" | None (bionic/msys) → CDLL(None)
    kernel_view_root: Path | None # proot host-side root; None elsewhere
    path_translator: Callable[[Path], Path] | None
    case_sensitive: bool
```

---

## Known limitations

These are surfaced as `partial` or `none`, and the planner refuses to hide them:

- **Windows `acl-token`** restricts *writes* only; reads follow the ambient token ACL. A hard link created inside the workspace can reference a file outside it, and deleting the link can affect the outside target.
- **WSL drvfs** (`/mnt/c`) writes are decided by NT ACLs independent of any Landlock ruleset — enforcement is `partial` for workspaces there.
- **Termux** may have Landlock in the kernel but filtered out by seccomp; only the syscall probe (with errno discrimination between `ENOSYS`, `EOPNOTSUPP`, and `EPERM`) tells the truth. `proot` is a ptrace path-rewriter, **not a security boundary**, and never satisfies a strength floor on its own.
- **Slurm / HPC** enforcement is `partial` until the workspace is validated as visible on the compute node; the scheduler boundary is not a filesystem boundary. The same applies to container images that exist on the login node but not on compute nodes.
- **`landlock`** cannot deny a single file inside an otherwise-writable directory; use bwrap mount ordering for that.
- **Network isolation** is out of scope for v1. The filesystem sandbox makes no claim about network access.
- **Container spawn permission** is only *provable* when a hello-world image is already present locally. Without one, the probe reports `unknown` and will not pull or build unless you opt in. This is a feature, not a gap.
- **Harbor** host-native environments cannot run `docker-compose.yaml`, build task images, or provide GPU/GUI passthrough; those tasks are rejected up front. Single-container tasks are allowed only when the image is already present and `use_existing=true`.

---

## Status

Pre-release. Planned sequencing:

- **v1** — Linux, local transport only. Probe + CLI + planner + `bwrap`/`landlock`/`podman`/`docker` backends, fail-closed. CI matrix runs inside containers, which doubles as the container-detection test suite.
- **v1.1** — `HarnessSandboxEnvironment` for Harbor (Linux + Windows local transport), including `min_strength`, task gating, and `ContainerSpawnPolicy`.
- **v1.2** — Broader spawner coverage: `singularity`/`apptainer`, `nerdctl`, `crictl`, rootless `podman` with `newuidmap`, `nvidia-container-toolkit` annotation.
- **v2** — `SSHTransport` with the same remote probe script, including remote container-spawner detection.
- **v3** — Slurm / PBS / SGE with shared-filesystem validation and shell-compiled plans, plus a Slurm-backed Harbor environment and scheduler-mediated container spawn.
- **v4** — Windows ACL, macOS seatbelt, MSYS2, and Termux backends.

The Transport seam exists from v1 so that v2 is an addition, not a rewrite. The Harbor integration is a *consumer* of `plan()`/`confine()`, not a new backend or transport.

---

## Development

```bash
git clone https://github.com/<org>/harness-sandbox
cd harness-sandbox
uv sync --all-extras
uv run python -m harness_sandbox probe --pretty
uv run pytest
uv run pytest -m harbor      # Harbor integration tests
uv run pytest -m spawner     # container spawner detection tests
```

No existing sandbox framework is a dependency — the capability-aware routing is the point of the project. Harbor is an optional extra so the core stays thin. The container-spawner probe shells out to whatever OCI clients are on `PATH`; it never vendors a runtime.

---

## License

[AGPL](./LICENSE)

In short, if you modify and deploy it, you should also provide a source code with your modifications publicly on the internet. 

# template behavior

- compatible with uv, pdm, and micromamba
- auto devcontainer that show respect to python version
- set `DEPENDENCY_MANAGER=uv`, `DEPENDENCY_MANAGER=pdm`, or `DEPENDENCY_MANAGER=micromamba` in `.env` to select the project dependency manager. If unset, projects with a root `notebooks/` directory use micromamba and other projects use uv.
- bun is installed to execute npm tools by bunx
- the selected manager syncs its root manifest directly into `.venv`; if that manifest is missing, it is initialized from `requirements.txt`. The other manager's manifest is generated through a temporary project.

## requirements

`requirements.txt` lists the dependencies. Its first line may set the Python
version with `python-must`:

```text
python-must >= 3.10
numpy
# tools
ruff
black
```

A comment that holds nothing but a name opens a dependency group, and every
requirement after it belongs to that group: `numpy` above is a project
dependency, while `ruff` and `black` are in the `tools` group. Other comments
and blank lines are ignored. Each group is installed by the manager itself,
as a uv dependency group or as a pip install into the micromamba environment.

`requirements-dev.txt` is optional, is a plain list without groups, and is
installed as development dependencies: `jupyterlab` lives there, because it is
not needed to run the project.

## generated manifests

`pyproject.toml` and `environment.yml` are generated from the name of the
project root, so this template does not commit them. Each project gets its own,
and neither is ignored, so they can be committed or edited by hand per project.

## badges

Everything between these two markers is generated, so the badges follow the
project instead of a hardcoded repository:

```text
<!-- BEGIN BADGE INJECTION -->
<!-- END BADGE INJECTION -->
```

The injector reads the `origin` remote, the current branch, `python-must`,
`DEPENDENCY_MANAGER`, `LICENSE`, `.github/workflows/`, and the requirements
files, then writes the repository, Python, manager, Codespaces, Binder, CI,
license, and dependency badges. Leave the markers in place: without both of
them the README is left alone, and everything outside them is preserved.

## Binder

`binder/environment.yml` configures [Binder](https://mybinder.org): repo2docker
builds the project with its Miniconda image. It is a symlink to the generated
root `environment.yml`, so the environment is defined once.

The symlink is expected to dangle in a fresh clone: the root `environment.yml`
is generated per project and is not committed. The dependency watcher creates
the link once the root file exists, replaces a link that points elsewhere, and
leaves a regular file alone, since that one may hold hand edits.

Because the environment comes from the generated file, its Python version is
the one derived from `python-must`, so keep that line on a version repo2docker
ships a lock file for.

A Binder workspace is recognised by the `BINDER_*` variables that Binder sets.
In one, the dependency manager is always micromamba: a root `notebooks/`
directory is created so that the default resolution agrees, and the
requirements are installed into the conda environment that Binder already
built, instead of into a project `.venv`. `DEPENDENCY_MANAGER` cannot override
this, because uv cannot manage a conda environment.

Note that `notebooks/` is tracked once created, so a Binder visit can leave an
untracked directory behind in a checkout.

## tokens

`.env` also holds `GITHUB_TOKEN`, a personal access token with which the
scripts can manage the remote CI configuration. `git` ignores `.env`, and
`.env.example` carries a placeholder instead of a real token, so copy that file
and fill in the value:

```bash
cp .env.example .env
chmod 600 .env
```

## continuous integration

`.github/workflows/devcontainer.yml` runs `post-install.sh` and a one-shot
synchronization on a GitHub runner for each dependency manager, then checks the
generated manifest and badges. Use it instead of running the scripts locally
when the network is unreliable: `gh workflow run devcontainer.yml`, or push
under `.devcontainer/`, `binder/`, or `requirements*.txt`.

# publishing

The template is not a Python package: it describes an environment through
`requirements.txt`. To publish one anyway, `.devcontainer/build-package.sh`
generates a package in a scratch directory with `uv init --package`, which
writes a `pyproject.toml` and a `src` layout, and adds both requirements files
as its development dependencies, so the package and the environment cannot
drift apart. `python-must` and the `# <name>` group headers are dropped on the
way, since neither is a package.

```bash
bash .devcontainer/build-package.sh "$TMPDIR/mypython-template" fufurobot/mypython-template
# artifacts in $TMPDIR/mypython-template/dist
```

`.github/workflows/release.yml` runs that script and publishes to TestPyPI
through a [Trusted Publisher](https://docs.pypi.org/trusted-publishers/): the
workflow asks GitHub for an OIDC token, so no API token or password is stored.
The pending publisher on TestPyPI has to name, exactly:

| Field | Value |
| --- | --- |
| Project | `mypython-template-fufurobot` |
| Owner | `fufurobot` |
| Repository | `mypython-template` |
| Workflow | `release.yml` |
| Environment | `testpypi` |

Publishing happens on a `v*` tag or by running the workflow by hand:

```bash
git tag v0.1.0 && git push origin v0.1.0
gh workflow run release.yml
```

TestPyPI refuses a version that already exists, so every publish needs a new
one. A tag only triggers the workflow: it does not set the version. The version
is whatever `uv init` wrote into the generated `pyproject.toml`, so bumping it
between publishes is what makes each run uploadable.

### attestations

#### what they are

An attestation is a signed statement about an uploaded file. When the package
is published, the workflow can also send a small in-toto document per file that
says, in effect, *this exact wheel, with this exact hash, was built and
published by this specific GitHub Actions workflow of this repository*.

The signature does not come from a stored key. GitHub Actions mints a
short-lived OIDC token, Sigstore exchanges it for a certificate, the attestation
is signed with that certificate's key, and the signature is recorded in
Sigstore's transparency log, [Rekor](https://www.sigstore.dev/), so it can be
verified later by anyone, not just by PyPI.

These are the same OIDC mechanics that a Trusted Publisher uses for
authentication, reused to make a claim that outlives the upload. PyPI stores
the attestations it receives next to the files it serves, and they can be
checked afterwards with tooling such as `pypi-attestations` and
`pip install --attest` (see [PyPI's attestation
docs](https://docs.pypi.org/attestations/) and
[PEP 740](https://peps.python.org/pep-0740/)).

### why they matter for PyPI

- **Provenance.** A consumer can check that a wheel really came from this
  repository's release workflow, not from someone who rebuilt it, or uploaded a
  lookalike to another index.
- **Tamper evidence.** The signature binds the file contents to a hash, so a
  differ-by-one-byte substitution no longer goes unnoticed.
- **A durable audit trail.** The Trusted Publisher token that authorised the
  upload expires in minutes; the attestation remains, and names the workflow
  and repository that published it.
- **They are what PyPI is moving toward.** PyPI verifies publisher identity at
  upload time regardless, but attestations are the part that a downstream user
  can verify for themselves, without trusting the index's word for it.

None of this makes an upload work or fail, which is why turning them off is a
real choice rather than a break: the package still installs, it just arrives
without a verifiable statement about where it came from.

#### why they are off on TestPyPI

The publish step runs with `attestations: false` for one concrete reason:
Rekor answered `502 Bad Gateway` to this repository's runners, twice in a row,
while being healthy when probed from elsewhere. Because signing happens before
the upload, the failure aborted the publish and nothing reached TestPyPI at
all:

```text
sigstore._internal.rekor.RekorClientError: Rekor returned an unknown error with HTTP 502
```

TestPyPI is a rehearsal target: it exists to prove that the packaging and the
publisher configuration work, its uploads are disposable, and nobody installs
from it expecting a provenance guarantee. Taking on a dependency on a third
party's uptime, to add signatures nobody consumes there, is the wrong trade.

For PyPI proper the opposite holds, and the line should be dropped:

```yaml
- uses: pypa/gh-action-pypi-publish@release/v1
  with:
    # attestations: false        # remove this for PyPI proper
```

Before relying on it, confirm that Rekor is answering, since a `502` there
fails the publish before any upload starts.

#### artifacts survive a failed publish

The `build` job is separate from `publish`, so a failed publish never loses the
work: the wheel and sdist remain attached to the run as the `dist` artifact,
and only the `publish` job needs to be re-run:

```bash
gh run rerun <run-id> --failed
```

### what a release publishes

`.github/workflows/release.yml` runs four jobs. `build` generates the package,
and the other three fan out from it:

| Job | Output | Needs |
| --- | --- | --- |
| `build` | wheel and sdist, kept as the `dist` artifact | — |
| `publish` | the package on TestPyPI | `id-token: write` (OIDC) |
| `image` | the devcontainer image, on the GitHub Container Registry | `packages: write` |
| `release` | a GitHub release with the artifacts | `contents: write` |

Every run does all four, whether it was triggered by a `v*` tag or by hand.
A tag push names the release after the tag; any other run names it after the
version the build published, `v` plus that pre-release, so the artifacts of a
rehearsal are kept rather than discarded. Re-running a version replaces its
assets instead of failing.

## versions and collisions

An index refuses a version it already holds, so a second publish of `0.1.0`
fails. The obvious fix, tagging each build with its commit as a local version
(`0.1.0+2f7369b`), does not work: PEP 440 forbids a local version on a public
index, and both PyPI and TestPyPI reject it with

```text
400 The use of local versions in <Version('0.1.0+2f7369b')> is not allowed.
```

A pre-release segment may carry the commit instead, so the build job numbers
each build `BASE.devN`, where `N` is the decimal value of the commit's short
SHA: `0.1.0.dev49755803`. `.dev` accepts only a number, which is why the hex
digits are read as a decimal. That is unique per commit, but it does **not**
sort chronologically — a later commit can have a smaller number — so it is a
stopgap for keeping rehearsals from colliding, not a versioning policy.

For versions that mean something, bump them on purpose with
[bump-my-version](https://pypi.org/project/bump-my-version/), the tool listed
in `requirements-dev.txt`. It is worth using because it rewrites the version in
every file that carries it and commits the change, so the number is chosen
rather than derived, it sorts correctly, and it can produce a matching tag:

```bash
uv run bump-my-version bump patch   # 0.1.0 -> 0.1.1
uv run bump-my-version bump minor   # 0.1.1 -> 0.2.0
```

Configure it in `pyproject.toml` with the files that hold the version, and
point it at your tags:

```toml
[tool.bumpversion]
current_version = "0.1.0"
commit = true
tag = true
tag_name = "v{new_version}"
```

Once a version is chosen that way, the commit-derived pre-release is noise; the
build job derives it only because nothing else supplies a number.

### the devcontainer image

`.devcontainer/Dockerfile` builds the development environment as a standalone
image. It starts from mise's official Debian image,
`ghcr.io/jdx/mise:debian`, which is published for `linux/amd64` and
`linux/arm64` and already carries mise, curl, git, and CA certificates; the
earlier Arch Linux base was published for `linux/amd64` alone. On top of it the
Dockerfile creates the `vscode` user, runs `post-install.sh` so uv, pdm,
micromamba, and bun are installed, and fails the build if any of them cannot
run. It is pushed to `ghcr.io/fufurobot/mypython-template` for both platforms.

`devcontainer.json` builds that Dockerfile rather than pulling the published
image, so a fork works without depending on an image this repository pushes.
The image also runs mise's directories from `/mise`, outside the home
directory, so mounting `~` does not hide anything installed during the build.

The workflow authenticates with the built-in `GITHUB_TOKEN`, whose
`packages: write` permission comes from the job, so no token has to be stored
for automation. `GITHUB_PACKAGES_TOKEN` in `.env` exists for pushing from a
workstation, where that token is not available:

```bash
set -a && . ./.env && set +a
echo "$GITHUB_PACKAGES_TOKEN" | docker login ghcr.io -u fufurobot --password-stdin
docker build -f .devcontainer/Dockerfile -t ghcr.io/fufurobot/mypython-template:dev .
docker push ghcr.io/fufurobot/mypython-template:dev
```

To develop inside the published image instead of building it, point
`devcontainer.json` at it, keeping the mount of this repository:

```json
{ "image": "ghcr.io/fufurobot/mypython-template:latest" }
```

The image does not include the project: it carries the tools, and the
workspace is mounted over `/workspaces` as usual.
