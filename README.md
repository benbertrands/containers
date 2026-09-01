# containers

Container images for tools that do not publish one themselves, built here and
pushed to `ghcr.io/benbertrands`.

| Image | Upstream | Why it is built here |
| --- | --- | --- |
| [`claude-code`](claude-code/Dockerfile) | [`@anthropic-ai/claude-code`](https://www.npmjs.com/package/@anthropic-ai/claude-code) (npm) | Anthropic ships a devcontainer feature and a reference `.devcontainer`, both described upstream as a working example rather than a maintained base image. |
| [`obsidian-headless`](obsidian-headless/Dockerfile) | [`obsidian-headless`](https://www.npmjs.com/package/obsidian-headless) (npm) | Ships as an npm package and nothing else. |

Both tools install with `npm install -g`, which makes the installed version
whatever npm resolved that day. Building them into images instead means a
deployment can pin a version rather than resolve one at deploy time.

## Pulling

```sh
docker pull ghcr.io/benbertrands/obsidian-headless:0.0.14
docker pull ghcr.io/benbertrands/claude-code:2.1.239
```

These are personal builds, published publicly because there is no reason not to.
They track upstream releases and carry no patches — but they come with no support
or stability promise, so pin a version rather than tracking whatever is newest.

## Tags

**The tag is the upstream version, verbatim.** Nothing is appended: the image
tagged `0.0.14` contains `obsidian-headless@0.0.14`. There is no `latest`.

Tags are **mutable**. Rebuilding an unchanged upstream version republishes the
same tag pointing at a new digest — most often because the base image underneath
has moved. If you need byte-identical content across deployments, pin by digest:

```sh
docker pull ghcr.io/benbertrands/claude-code@sha256:<digest>
```

There is no scheduled rebuild. A container runtime does not re-pull a tag it
already has, so republishing a tag on a patched base would not reach anyone who
had already pulled it; base image updates ride along with the next version bump,
which changes the tag and is therefore actually fetched.

## How an update flows

The `ARG <TOOL>_VERSION=` line in each Dockerfile is the single source of truth.
Renovate bumps it, and merging that runs
[`images.yml`](.github/workflows/images.yml), which reads the same ARG to name
the tag and pushes `ghcr.io/benbertrands/<image>:X.Y.Z`. No second edit anywhere.

Renovate runs weekly from [`renovate.yml`](.github/workflows/renovate.yml) —
self-hosted, on the built-in `GITHUB_TOKEN`, so no third-party app access and no
long-lived PAT. It manages two things per image: the npm version, and the `FROM`
line.

Two consequences of using `GITHUB_TOKEN` are worth knowing:

- A pull request it opens does not trigger other workflows, so the images
  workflow does not build Renovate branches automatically. Push an empty commit
  to the branch if you want the build first.
- It cannot modify files under `.github/workflows/`, so **GitHub Action updates
  are disabled** in `renovate.json` rather than left to fail silently. The action
  pins are bumped by hand.

## One-time setup

GitHub Packages creates each package on first push, and visibility is set on the
package rather than inherited from this repository. After the first successful
run, for **each** of the two packages:

> github.com/users/benbertrands/packages/container/`<image>`/settings →
> **Change visibility** → **Public**

This is not exposed by the REST API, so it cannot be done from the workflow. It
sticks once set.

## Building locally

```sh
docker build -t obsidian-headless:dev obsidian-headless
docker build -t claude-code:dev claude-code
```

Nothing in either build is specific to CI; the workflow only adds the tag
derivation and the push.

## License

The Dockerfiles and workflows in this repository are MIT-licensed — see
[LICENSE](LICENSE). The software they install is covered by its own upstream
license.
