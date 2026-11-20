# libfprint patch for the ELAN 04f3:0903 96×96 press sensor

Fixes the ELAN `04f3:0903` fingerprint reader (ASUS VivoBook/Zenbook laptops,
firmware `0x0170`) under Linux: enrollment no longer aborts with
`Calibration failed!` / `protocol error`, and `fprintd-verify` returns
`verify-match` instead of `verify-no-match`.

Built on upstream libfprint **v1.94.100**, with the press-mode + SIGFM work
adapted from the AUR package [`libfprint-elan-0c28-patch`](https://aur.archlinux.org/packages/libfprint-elan-0c28-patch)
(which targets the sibling `04f3:0c28`), plus support for this PID.

## Symptoms this fixes

```text
fprintd[..]: Device reported an error during enroll: The driver encountered a protocol error with the device.
fprintd[..]: Device reported an error during identify for enroll: Calibration failed!
fprintd-verify: Verify result: verify-no-match (done)
```

…while `journalctl -k` shows **no** USB disconnect for the reader: the device
never leaves the bus, libfprint just gives up and deactivates it, which looks
like the scanner "switching off" after taking a print.

## Root cause

1. **Wrong capture model.** `04f3:0903` is a 96×96 *press-and-hold* sensor, but
   upstream `elan.c` registers it as a generic swipe device
   (`ELAN_ALL_DEV`) and stitches frames from finger movement. Dumping the
   assembled `FpImage` shows the result: a 240×1100 strip with low contrast and
   no ridge detail. Minutiae extraction finds nothing, so Bozorth3 always scores
   `0/10` — no threshold can rescue that.
2. **Calibration starts with a finger on the sensor.** `0x4023` calibration is
   driven from the background frame; a resting finger makes it dark
   (`bg mean ≈ 900` instead of `≈ 2900`), the status byte stays `0x01`, the
   attempt counter runs out and the action dies with `Calibration failed!`.
3. **`0xaf` treated as fatal.** `pre_scan` returning `0xaf` means "finger
   removed / sensor busy". Upstream marks it `FP_DEVICE_ERROR_PROTO` and aborts
   mid-enroll instead of waiting for the next touch.

## What the patch does

* `ELAN_PRESS` device class for `04f3:0903` (and configurable for other square
  sensors): a single sharpest/averaged full frame instead of swipe stitching;
  stripe-noise rejection via a column-variance ratio; 15 enroll stages.
* **SIGFM** matching (`libfprint/drivers/sigfm.c`, FAST-9 + BRIEF-256 +
  RANSAC) instead of NBIS/Bozorth3, which is far more tolerant of tiny images.
  Keypoints are stored as serialized blobs in the print's RAW data.
* Calibration hardening: retry rounds when a finger lands during calibration,
  wait for a clean background, and much longer status-polling windows.
* `0xaf` from `pre_scan` re-enters the wait-for-finger state instead of failing.
* `tests/meson.build`: `foreach x, y: dict` so the tree configures with
  meson ≥ 1.12.

## Install (Arch)

```bash
makepkg -f
sudo pacman -U libfprint-elan-0903-patch-1.94.100-1-x86_64.pkg.tar.zst
sudo systemctl restart fprintd
fprintd-enroll "$USER"     # touch, hold ~1 s, lift; repeat (~15 + a couple extra)
fprintd-verify "$USER"
```

The package conflicts with `libfprint`/`libfprint-git` and provides
`libfprint-2.so=2-64`, so `fprintd` keeps working. It does **not** ship GIR
typelibs or gtk-doc (built with `-Dintrospection=false -Ddoc=false`).

### Build without touching the system

Only `meson`, `ninja` and the glib tools are needed. If `glib2-devel` cannot be
installed, unpack it and point pkg-config at it:

```bash
curl -LO https://geo.mirror.pkgbuild.com/core/os/x86_64/glib2-devel-<ver>-x86_64.pkg.tar.zst
mkdir -p tools && bsdtar -xf glib2-devel-*.pkg.tar.zst -C tools usr/bin
cp /usr/lib/pkgconfig/glib-2.0.pc pc/
sed -i "s#^bindir=.*#bindir=$PWD/tools/usr/bin#" pc/glib-2.0.pc
export PATH="$PWD/tools/usr/bin:$PATH" PKG_CONFIG_PATH="$PWD/pc"
```

## Usage notes

* **Touch, don't swipe**, and hold ~1 second. The pad is ~8×8 mm, so touching
  the same spot with the same finger matters more than anything else: with
  loose placement genuine scores dropped to 4–5 (below threshold 7), with a
  centred firm press they were 21–27.
* `ELAN_SIGFM_THRESHOLD` (7) is deliberately conservative — impostor scores on
  a comparable sensor reached 6. Lower it only if you accept the trade-off.
* Tuning knobs live in `libfprint/drivers/elan.h`: `ELAN_PRESS_ENROLL_STAGES`,
  `ELAN_PRESS_FRAMES`, `ELAN_PRESS_MAX_STRIPE_RATIO`,
  `ELAN_PRESS_CALIBRATION_ROUNDS`, `ELAN_SIGFM_MIN_KEYPOINTS`.
* `fprintd-list` reports the reader as `(press)`. Enroll emits
  `enroll-stage-passed` N times and may ask for a few extra touches before
  `enroll-completed`; those are not failures.

## Extra pieces (optional)

* `udev/99-elan-0903-no-autosuspend.rules` — keeps the reader out of USB
  autosuspend; mid-enroll suspends break captures.
* `scripts/90-fprint-patch-check.hook` + `scripts/fprint-patch-check` — pacman
  `PostTransaction` hook that checks the installed library for the patch marker
  (`sigfm best score`), restarts `fprintd` when it is present, and shouts (log +
  desktop notification) if a stock `libfprint` has replaced it.

```bash
sudo install -m 755 scripts/fprint-patch-check /usr/local/bin/
sudo install -m 644 scripts/90-fprint-patch-check.hook /etc/pacman.d/hooks/
sudo install -m 644 udev/99-elan-0903-no-autosuspend.rules /etc/udev/rules.d/
```

## Packaging note for Omarchy

Omarchy's `omarchy setup security fingerprint` refuses to run unless a package
named `libfprint-git` is installed and would otherwise pull stock libfprint over
this build. Locally the package is therefore built under the name
`libfprint-git` with a version higher than the AUR one. Upstream this repo uses
the neutral name `libfprint-elan-0903-patch`; rename it in your own build if your
tooling has the same check.

## Debugging a similar sensor

`fprintd.service` sets `PrivateTmp=true`, so dumps written to `/tmp` by a driver
land in the service namespace — read them via `/proc/<fprintd-pid>/root/tmp/`.
Turning a debug build on is just:

```bash
sudo mkdir -p /etc/systemd/system/fprintd.service.d
printf '[Service]\nEnvironment=G_MESSAGES_DEBUG=all\n' | sudo tee /etc/systemd/system/fprintd.service.d/debug.conf
sudo systemctl daemon-reload && sudo systemctl restart fprintd
journalctl -b -u fprintd -f
```

Useful lines: `calibration mean: X, bg mean: Y`, `calibration status: 0xNN`,
`pre_scan status 0xNN`, `press image: sharpest of N frames`, `sigfm keypoints`,
`sigfm best score N/7`.

## License

LGPL-2.1-or-later, same as libfprint (`LICENSE`). See `CREDITS.md`.
