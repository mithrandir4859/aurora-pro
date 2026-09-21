# aurora-pro

A personal [Aurora DX](https://getaurora.dev/) image: stock `ghcr.io/ublue-os/aurora-dx:stable` with a few packages baked in, built by [BlueBuild](https://blue-build.org/) and published to `ghcr.io/mithrandir4859/aurora-pro`.

## Why this exists

Some packages were layered with `rpm-ostree install`, and some can't be layered on Aurora at all.

- **Steam can't be layered.** Aurora edits the default icon theme, so layering `libXcursor` fails with a hardlink error (`ublue-os/bluefin#1258`, closed "not planned"). A container build has no ostree hardlinking, so it installs cleanly.
- **Wine can't be layered either.** The i686 packages hit the same class of hardlink failure (RHBZ #1846803, rpm-ostree#1937). A system-wide `wine foo.exe` is only possible from an image.
- **Layering has a recurring cost.** It slows every update, can block upgrades and must be cleared before a major rebase. An image costs a fixed setup up front, and then each extra package costs nothing.

It derives from the upstream image instead of forking `ublue-os/aurora`, so there are no merge conflicts to chase. A daily build picks up Aurora updates automatically.

What belongs in the image: things that need to write to `/usr`, own a system-wide default, or load into the kernel. GUI apps go in Flatpak, CLI tools in Homebrew, and everything else in Distrobox.

## What's in it

See `recipes/recipe.yml`, which also explains each choice in its comments.

- `kvantum` (Qt style engine) and `snapper` (btrfs snapshots).
- Gaming, from negativo17 (the same repo family Aurora's Mesa comes from, so the 32-bit stack matches): `steam`, `steam-devices`, `gamescope`, `mangohud`.
- Wine as a host tool: `wine`, `wine-core.i686`, `wine-pulseaudio.i686`, `wine-mono`, `winetricks`, `lutris`.
- The BlueBuild `signing` module, so machines verify the image's signature.

## Builds and tags

GitHub Actions (`.github/workflows/build.yml`) builds on every push, on pull requests, on manual dispatch, and daily at 06:00 UTC. The scheduled build runs only on `main`.

| Tag | Meaning |
|---|---|
| `latest` | Newest build of `main`. Follow this one. |
| `44`, `20260921`, `20260921-44` | Fedora version and build date, from `main`. |
| `<sha>-44` (e.g. `8e408e3-44`) | A specific commit's build. Pin to it to go back to a known-good image. |
| `br-<branch>-44` (e.g. `br-steam2-44`) | Latest push to a non-default branch, for trying changes before merging. It is **not** rebuilt daily, so don't stay on it. |

The build keeps the Fedora version from the base image, so `latest` won't jump to a new Fedora release until Aurora `stable` does.

## Installing on a machine

The machine's `/etc/containers/policy.json` rejects signed images from unknown namespaces. The image carries its own trust policy, so the first rebase is unverified, and after that everything is signed. This is needed once per machine, not per tag.

```bash
# optional: verify the image out-of-band first (cosign.pub is in this repo)
cosign verify --key cosign.pub ghcr.io/mithrandir4859/aurora-pro:latest

rpm-ostree rebase ostree-unverified-registry:ghcr.io/mithrandir4859/aurora-pro:latest
systemctl reboot

rpm-ostree rebase ostree-image-signed:docker://ghcr.io/mithrandir4859/aurora-pro:latest
systemctl reboot
```

Remove any layered packages the image now provides, for example `rpm-ostree uninstall kvantum`, before or after the rebase.

## Day to day

```bash
# what am I running, and what's staged
rpm-ostree status

# switch tag (a branch build, a pinned commit, back to latest)
sudo rpm-ostree rebase ostree-image-signed:docker://ghcr.io/mithrandir4859/aurora-pro:latest
sudo rpm-ostree rebase ostree-image-signed:docker://ghcr.io/mithrandir4859/aurora-pro:8e408e3-44

# update now instead of waiting for uupd.timer (Aurora's updater, runs daily)
sudo rpm-ostree upgrade

# boot the previous deployment (also selectable in GRUB)
sudo rpm-ostree rollback

# back to stock Aurora
sudo rpm-ostree rebase ostree-image-signed:docker://ghcr.io/ublue-os/aurora-dx:stable
```

Checking builds and published images:

```bash
gh run list -R mithrandir4859/aurora-pro        # or the repo's Actions tab
gh run watch -R mithrandir4859/aurora-pro

skopeo list-tags docker://ghcr.io/mithrandir4859/aurora-pro
skopeo inspect docker://ghcr.io/mithrandir4859/aurora-pro:latest   # build date, Aurora version, digest

# what a build actually contains
podman run --rm ghcr.io/mithrandir4859/aurora-pro:latest rpm -q steam wine lutris
```

## Changing the image

Edit `recipes/recipe.yml` on a branch and push. The branch builds as `br-<branch>-44`; rebase one machine to it, try it, then merge into `main`. Files placed under `files/system/` are copied into the image root.

Before adding a package, check it against the live package set, for example `dnf5 install --assumeno <pkg>` in a container of `aurora-dx:stable`, so the dependency count holds no surprises.

## Signing

Images are signed with [cosign](https://github.com/sigstore/cosign). `cosign.pub` is committed. The private key is the `SIGNING_SECRET` repository secret; `cosign.key` is gitignored and must never be committed.
