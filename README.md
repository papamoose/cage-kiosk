# cage-kiosk

Builds arm64 Debian .deb packages for a Raspberry Pi 5 kiosk.

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
cage/               cage source (upstream + kiosk options) with debian/
wlroots-debian/     debian/ packaging copied over the wlroots tarball
patches/            the kiosk patch, for reference and upstreaming
```

## Building

GitHub Actions (`.github/workflows/build-debs.yml`) builds on every
push to `main` and on tags:

1. Runs a Debian trixie container with `--platform linux/arm64`
   (QEMU-emulated on the x86 runner).
2. Downloads the wlroots tarball pinned in `WLROOTS_VERSION`, applies
   `wlroots-debian/`, and runs `dpkg-buildpackage`.
3. Builds `cage/` the same way.
4. Uploads the .debs as an artifact. On a tag (`v*`), creates a
   GitHub release and attaches the .debs.

Version pins live in the workflow `env` block. To bump wlroots or
cage, change the pin and the matching `debian/changelog` entry, then
push.

Local build (needs an arm64 machine or container with the trixie
build dependencies from the workflow):

```
dpkg -i libwlroots-0.20_*.deb libwlroots-0.20-dev_*.deb
(cd cage && dpkg-buildpackage -us -uc -b)
```

## Installing on a kiosk

```
apt-get install ./libwlroots-0.20_*.deb ./cage_*.deb
```

`cage` depends on `libwlroots-0.20`, so both must come from this
repository. The stock trixie `cage` (0.2.0, wlroots 0.18) is replaced.
