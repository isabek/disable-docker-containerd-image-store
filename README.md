# Disable Docker containerd image store

A GitHub Action that turns off Docker's containerd image store (`containerd-snapshotter`) on Linux runners and restarts `dockerd`, so Docker goes back to its classic image store and storage driver (usually `overlay2`).

## Why

The containerd image store is the default since Docker Engine 29. Some tools don't work with it yet, notably Cloud Native Buildpacks as used by Spring Boot's `bootBuildImage` / `spring-boot:build-image`. Typical errors:

- `failed to export: saving image: failed to fetch base layers ... No such image`
- `Error response received when pushing image: error from registry: manifest invalid`

See [spring-projects/spring-boot#49251](https://github.com/spring-projects/spring-boot/issues/49251) and [buildpacks/lifecycle#1605](https://github.com/buildpacks/lifecycle/issues/1605).

This action is a workaround until those tools support the containerd image store.

## Usage

```yaml
steps:
  - uses: OWNER/disable-docker-containerd-image-store@v1
  - uses: actions/checkout@v4
  - run: ./gradlew bootBuildImage --publishImage
```

Run it **before** any other Docker-related step in the job.

## What it does

1. Checks whether Docker uses the containerd image store. If not (e.g. Docker 28 runners), it does nothing and doesn't restart Docker.
2. Otherwise sets `features.containerd-snapshotter` to `false` in `/etc/docker/daemon.json`, keeping any existing settings.
3. Restarts `dockerd` and fails the step if the containerd image store is still active afterwards.

## Outputs

| Name             | Description                                                                 |
|------------------|-----------------------------------------------------------------------------|
| `docker-version` | Docker server version after the change                                      |
| `storage-driver` | Storage driver in use after the change (e.g. `overlay2`)                    |
| `restarted`      | `true` if `dockerd` was restarted, `false` if the containerd store was already off |

## Caveats

- **Restarting `dockerd` stops running containers.** Job-level `services:` containers (databases, etc.) are started before your steps and won't come back. If you need them, start them yourself after this step.
- **Images pulled or built earlier in the job become invisible**, because they live in the containerd image store. Run this action first.
- **Self-hosted runners:** the change to `/etc/docker/daemon.json` persists after the job and affects every later job on that host.
- Requires a Linux runner with a systemd-managed Docker daemon, `sudo` and `jq`. GitHub-hosted Ubuntu runners have all of these. It doesn't work in `container:` jobs or on macOS/Windows runners.

## License

[MIT](LICENSE)
