# deven image builder

Builds the **deven** container image — the development-environment base of the
[Pi](https://pi.dev/) agent image family — and publishes it to Alibaba Cloud
ACR via the ACR builder service.

This is one of three per-image repositories split out of the archived
[pi-agent-image](https://github.com/lulin/pi-agent-image) project:

```
deven  ->  pi-vanilla  ->  pi-agent
```

One repository per image because the ACR builder service has no workflow
orchestration: a dedicated repo gives each image exactly one builder, and the
chain is enforced by consuming parent images **by tag from ACR**.

## What is in the image

| Layer | Contents |
|-|-|
| Base | Ubuntu 24.04 LTS (glibc) + pi's runtime tools: bash, git, curl, jq, ripgrep, fd, openssh-client, vim, iputils; + native-build tooling: build-essential, pkg-config, libssl-dev |
| Toolchain | Node.js 24 (official tarballs into `/usr/local`), uv + managed CPython 3.14 (`/opt/uv/python`), Rust via rustup (`/opt/rustup`, `/opt/cargo`, `PATH` extended, clippy/rustfmt) |

The managed CPython has no pip — use `uv pip` / `uv add` / `uv run`.
`python` / `python3` / `python3.14` are symlinked onto the PATH.

## Repository layout

```
Dockerfile    # multi-stage: base -> toolchain -> deven (from pi-agent-image/aliyuncs/deven)
```

## Building

Builds run in the **ACR builder service** (云端构建), one rule for this repo:

| Setting | Value |
|-|-|
| Code source | this GitHub repository |
| Dockerfile path | `Dockerfile` |
| Context directory | repo root (no context files are used) |
| Namespace/repo | `***/deven` |
| Tag rules | `latest`; `v{major}` etc. as preferred |
| Build args | none required — defaults are correct |

Optional build args (all default to upstream sources; ACR builders have
overseas access): `APT_MIRROR=none`, `NODE_VERSION=24`, `NODE_DIST_URL`,
`UV_VERSION`, `UV_INSTALLER_GITHUB_BASE_URL`, `UV_PYTHON_INSTALL_MIRROR`,
`PYTHON_VERSION`, `RUST_VERSION`, `RUSTUP_DIST_SERVER`, `RUSTUP_UPDATE_ROOT`.

Manual build anywhere with a docker daemon:

```bash
docker build -t deven:latest .
```

## After building

Rebuild dependents **in order**: `deven` → `pi-vanilla` → `pi-agent`. They pull
`registry.cn-hangzhou.aliyuncs.com/***/deven:latest` as their parent image;
same-region builders may prefer the VPC endpoint `registry-vpc.cn-hangzhou...`.

## Pulling

```bash
docker pull registry.cn-hangzhou.aliyuncs.com/***/deven:latest
docker run -it --rm -v "$PWD:/workspace" -w /workspace \
  registry.cn-hangzhou.aliyuncs.com/***/deven:latest bash
```
