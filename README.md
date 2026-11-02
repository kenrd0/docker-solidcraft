# SolidCraft desktop on Selkies

Streams the **SolidCraft Linux desktop** to your browser using
`ghcr.io/linuxserver/baseimage-selkies:debiantrixie`. This is not SolidCraft's
separate web app. The image installs the upstream `.deb`, verifies its release
SHA-256 checksum, and supports `linux/amd64` and `linux/arm64` with one Dockerfile.

## Build and run

```sh
# Software rendering/encoding; no GPU required
docker compose up -d --build

# Intel or AMD GPU (use this instead of the command above)
docker compose -f compose.yaml -f compose.gpu.yaml up -d --build

# NVIDIA GPU (use this instead; see host requirements below)
docker compose -f compose.yaml -f compose.nvidia.yaml up -d --build
```

Open **<https://localhost:3001>** and accept the base image's self-signed certificate.
The port binds to loopback only: for a remote Docker host use an SSH tunnel,
e.g. `ssh -L 3001:localhost:3001 user@docker-host`. HTTP port 3000 is not published.

`/config` is a named volume containing settings and saved files. Save projects
there, or add a separate project volume. `docker compose down` preserves data;
`docker compose down -v` deletes it. Set `PUID`, `PGID`, and `TZ` in your shell or
`.env` to match your host. Do not override the base image's user or entrypoint;
it handles permissions and starts the application as the unprivileged `abc` user.

The SealSkin catalogue entry enables file handoff for SolidCraft design, CAD, and
mesh formats. Both X11 and Wayland startup scripts pass SealSkin's `SEALSKIN_FILE`
to SolidCraft so the selected file opens in the app. The image also includes
Zenity, the fallback SolidCraft's native file dialog uses when no desktop portal
is available in the container.

For network access, use an authenticated HTTPS reverse proxy or the base image's
`CUSTOM_USER` and `PASSWORD` settings. This desktop provides terminal access and
passwordless sudo inside the container; do not expose it directly to the Internet.

## GPU setup

Rendering SolidCraft and encoding the browser stream are separate GPU tasks.
Selkies auto-detects mounted GPUs. Mesa Vulkan/OpenGL libraries are included;
the Docker **daemon host** needs a working GPU driver. No privileged container,
host display socket, or FUSE is required.

### Verify and troubleshoot

```sh
docker compose logs --tail=200 solidcraft
docker compose exec solidcraft vulkaninfo --summary
# Intel / AMD encoder capabilities
docker compose exec solidcraft vainfo
# NVIDIA visibility (NVIDIA override only)
docker compose exec solidcraft nvidia-smi
```

Check the Vulkan device name: `llvmpipe`/`lavapipe` means software rendering, not
hardware acceleration. Check SolidCraft's graphics settings/adapter information
and startup logs too; a working encoder alone does not prove CAD rendering uses
the GPU. Test a model interactively before relying on acceleration.

SolidCraft normally selects its backend automatically. If necessary, add
`WGPU_BACKEND=vulkan` or `WGPU_BACKEND=gl` to the service environment to test a
specific backend. An explicit backend disables automatic fallback. The image
defaults to native Wayland; set `PIXELFLUX_WAYLAND=false` to test the X11 stack.
Both session types have autostart and right-click relaunch menu entries.

The base image copies launcher/menu defaults into `/config` **only on first
run**. Existing customized files are retained across rebuilds; use a separate
fresh volume to test changed defaults without deleting your projects.

## Versions and architectures

```sh
docker build --pull -t solidcraft:local -f docker/Dockerfile docker
# Override the pinned upstream release (without a leading v)
docker build --build-arg SOLIDCRAFT_VERSION=0.30.0 \
  -t solidcraft:0.30.0 -f docker/Dockerfile docker
```

Compose accepts the same version via `SOLIDCRAFT_VERSION`. Use `--pull --no-cache`
when rebuilding to pick up refreshed packages and the distro-tagged base image.
The version pin and upstream checksum protect against accidental mismatches;
the checksum is fetched from the same release, not an independent signature.

To publish both architectures, use a Buildx builder with native ARM64 workers or
QEMU/binfmt configured and replace the example registry below:

```sh
docker buildx build --platform linux/amd64,linux/arm64 \
  -t your-registry/solidcraft:0.30.0 -f docker/Dockerfile --push docker
```

The Forgejo workflow builds the runner's native architecture and pushes `latest`
and versioned tags to the configured registry. It verifies `solidcraft-cli`
without starting the desktop; GUI and GPU validation require runtime testing.

## References

- [SolidCraft downloads](https://github.com/tangency-labs/solidcraft/releases)
- [Selkies base images](https://docs.linuxserver.io/selkies/components/baseimages/)
- [Building custom Selkies images](https://docs.linuxserver.io/selkies/developer-guide/building-images/)
- [Selkies GPU acceleration](https://docs.linuxserver.io/selkies/user-guide/gpu/)
