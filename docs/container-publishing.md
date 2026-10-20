# Container images for this fork

This fork publishes `ghcr.io/0xkaz/artex` from its own checked-out source.
Images contain the embedded web UI, Linux backend, bundled skills, runtime
tools, license and README. No model API keys or database credentials are needed
to build an image; configure those on the machine running ARTEX.

## Build and publish

The `release` GitHub Actions workflow builds the frontend, cross-compiles the
backend, smoke-tests the Linux amd64 binary and image, and publishes a combined
`linux/amd64` + `linux/arm64` image. The ARM image is built, not runtime-tested.
Existing CI remains responsible for the application regression suites.

| Event | Container tags | GitHub Release |
| --- | --- | --- |
| Push to `main` | `latest`, `sha-<full commit SHA>` | No |
| Push a `v*` tag | Exact tag, `sha-<full commit SHA>` | Yes, platform ZIPs and checksums |
| Manual run on `main` | `latest`, `sha-<full commit SHA>` | No |
| Relevant pull request or manual run on another branch | Build and smoke test only | No |

`latest` tracks main. Publishing a version tag does not move `latest`.
Branch builds embed a `dev-<SHA>` version, so the in-app binary updater is
disabled for them. Tagged builds use releases from `0xkaz/ARTEX`.

After merging the workflow:

1. Open the repository's **Actions** tab. If GitHub disabled workflows when
   forking, enable them.
2. Run **release → Run workflow → main** if a main push did not start a run.
3. Confirm the frontend, binaries and docker jobs succeed. The release job is
   intentionally skipped on main.
4. Open the linked `artex` package to check the published tags and digest.

Publishing uses the workflow's `GITHUB_TOKEN` with `packages: write`.
Docker Hub credentials and a manually stored publishing PAT are not required.
Repository or organization Actions restrictions can still prevent publication.
If the package already exists, grant this repository Actions access to it.

GHCR packages are private on first publication. Keep that setting for owner-only
use, or explicitly change package visibility to public if anonymous pulls are
wanted. A public source repository does not itself guarantee anonymous pulls.
See [GitHub's registry documentation](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry).

## Run the published image

From a checkout of this fork:

```sh
cp .env.example .env
# Edit .env: set POSTGRES_PASSWORD and, optionally, an LLM API key.
# ARTEX_IMAGE defaults to ghcr.io/0xkaz/artex.
docker compose pull
docker compose up -d
```

Open `http://localhost:8787`. The management UI and traffic proxy are bound to
loopback. Remote management can use an SSH tunnel or an explicitly configured
reverse proxy.

For a private package, first log in with a GitHub PAT (classic) that has
`read:packages` and permission to read the package. Store it in `CR_PAT` securely:

```sh
printf '%s' "$CR_PAT" | docker login ghcr.io -u 0xkaz --password-stdin
```

Existing `.env` files must set `ARTEX_IMAGE=ghcr.io/0xkaz/artex` and choose a tag
that this fork has actually published. For a specific source revision, set
`ARTEX_TAG=sha-<full commit SHA>` from the successful run. Use the registry digest
when an immutable reference is required; rebuilding a commit can pick up newer
base images or runtime tools.

For updates, pull and recreate the container:

```sh
docker compose pull artex
docker compose up -d artex
```

Back up the database and `data/` first. For containers, prefer image replacement
over the in-app binary updater: replacing only the binary leaves the runtime
tools unchanged and is undone when the container is recreated.

The Compose file preserves the existing `./skills:/app/skills` bind mount.
That local directory overrides skills bundled in the image; keep its checkout
aligned with the image revision, or deliberately maintain it as a customization.

## Build locally

The Dockerfile packages a prebuilt binary; `docker build` alone does not compile
the application. With the Go version from `go.mod` and Node.js 22 installed:

```sh
./build.sh --target linux/amd64
mkdir -p dist/amd64
cp dist/artex-linux-amd64/artex dist/amd64/artex
docker build --build-arg TARGETARCH=amd64 -t artex:local .
ARTEX_IMAGE=artex ARTEX_TAG=local docker compose up -d
```

For a native ARM64 build, replace `amd64` with `arm64` throughout. Keep the build
platform and binary architecture consistent.
