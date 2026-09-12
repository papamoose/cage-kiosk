# cage-kiosk

Builds arm64 Debian .deb packages for a Raspberry Pi 5 kiosk to run Firefox which points at ImmichFrame.

This builds on top of [cage-kiosk/cage](https://github.com/cage-kiosk/cage) and is kept
as a submodule pinned to the upstream commit plus `patches/cage-kiosk-options.patch`,
which CI applies at build time. That keeps the kiosk changes a single 
reviewable patch on top of a clean upstream tree, so bumping
the submodule stays a one-line change.

Two packages come out of every build:

- **libwlroots-0.20** - wlroots 0.20 for Debian trixie arm64. The
  trixie and Raspberry Pi archives only ship wlroots 0.18, but current
  cage targets 0.20. Built from the pinned upstream tarball.
- **cage** - cage with kiosk options on top of upstream:
  - `-t <transform>` - output transform (fix a physically rotated panel)
  - `-T` - touch-only seat, no cursor
  - `-c X,Y` - initial cursor position (default: center)
  - input devices without an output name map to the sole output, so
    touch follows rotated outputs (combined display panels, e.g. the
    Raspberry Pi 10" DSI)

Verified on a Raspberry Pi 5 (2 GB, Debian 13 aarch64) with a 10" DSI
touch panel: landscape output via `-t 90`, working touch, and

```
cage -s -m last -t 90 -c 960,1199 -- firefox-esr --kiosk http://<url>/
```

## Repository layout

```
cage/               git submodule: upstream cage, pinned to a commit
cage-debian/        debian/ packaging copied into the cage tree at build time
wlroots-debian/     debian/ packaging copied over the wlroots tarball
patches/            the kiosk patch, applied over the submodule at build time
```

## Building

GitHub Actions (`.github/workflows/build-debs.yml`) builds on every
push to `main` and on tags:

1. Runs a Debian trixie container with `--platform linux/arm64`
   (native on the self-hosted arm64 runner).
2. Downloads the wlroots tarball pinned in `WLROOTS_VERSION`, applies
   `wlroots-debian/`, and runs `dpkg-buildpackage`.
3. Applies `patches/cage-kiosk-options.patch` over the `cage/`
   submodule, copies in `cage-debian/`, and builds the same way.
4. Uploads the .debs as an artifact. On a tag (`v*`), creates a
   GitHub release and attaches the .debs.

Version pins live in the workflow `env` block. To bump wlroots or
cage, change the pin and the matching `debian/changelog` entry, then
push. To bump the cage submodule, check out the new commit, rebase the
patch if the base moved, and commit the new submodule pointer.

Local build (needs an arm64 machine or container with the trixie
build dependencies from the workflow):

```
git submodule update --init
dpkg -i libwlroots-0.20_*.deb libwlroots-0.20-dev_*.deb
git -C cage apply patches/cage-kiosk-options.patch
cp -a cage cage-build && cp -a cage-debian/. cage-build/debian/
(cd cage-build && dpkg-buildpackage -us -uc -b)
```

## Installing on a kiosk

```
apt-get install ./libwlroots-0.20_*.deb ./cage_*.deb
```

`cage` depends on `libwlroots-0.20`, so both must come from this
repository. The stock trixie `cage` (0.2.0, wlroots 0.18) is replaced.
