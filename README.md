# libfprint — Goodix 538d driver

A focused fork of libfprint that adds a working Linux driver for the **Goodix
`27c6:538d`** fingerprint sensor (the `goodixtls53xd` driver, advertised as
"Goodix TLS Fingerprint Sensor 53XD"). Found in many Dell laptops (e.g. Inspiron
7506/5000 series). This device is **not** supported by upstream libfprint or by
Goodix on Linux.

This is intentionally a **single-device build**: the CI and recommended build
compile only the `goodixtls53xd` driver. It is not a general libfprint
replacement — it exists to make the 538d work.

## ⚠️ Prerequisite: flash the sensor firmware first

The driver hard-requires the community firmware `GF5298_GM168SEC_APP_13016`. You
must flash it once with [goodix-fp-dump](https://github.com/goodix-fp-linux-dev/goodix-fp-dump)
before this driver will work:

```sh
git clone --recurse-submodules https://github.com/goodix-fp-linux-dev/goodix-fp-dump
cd goodix-fp-dump
python -m venv .venv && . .venv/bin/activate && pip install -r requirements.txt
sudo .venv/bin/python run_538d.py     # confirm, then press your finger when prompted
```

Notes:
- The flash is **permanent** and goes through the device's IAP bootloader, so a
  failed run can be retried — you will not brick the sensor.
- **Recommended on Linux-only machines.** On a Windows dual-boot, Windows Hello
  may stop working or re-flash stock firmware.

## Build

Dependencies: `glib >= 2.56`, `gusb`, `gobject-introspection`, and `openssl`
(the TLS-PSK session). On Arch/CachyOS: `glib2 libgusb gobject-introspection openssl meson`.

```sh
meson setup _build -Ddrivers=goodixtls53xd -Dudev_rules=enabled
ninja -C _build
```

`-Ddrivers=goodixtls53xd` keeps the build to this one driver (and avoids pulling
pixman/nss/etc. that other drivers need). On a modern GCC (14+) add
`-Dc_args=-Wno-error=incompatible-pointer-types` — this is 1.94.1-era code.

## Test on hardware

```sh
sudo ./_build/examples/img-capture finger.pgm   # capture one image
sudo ./_build/examples/enroll                    # enroll a finger (5 presses)
sudo ./_build/examples/verify                    # verify against it
```

Press flat with full, even, light pressure. A good match scores well above the
`bz3_threshold` of 24 (≈33 in testing).

## Install for daily use

The GitHub Actions workflow (`.github/workflows/main.yml`) builds `.deb`, `.rpm`,
and Arch packages via [nFPM](https://nfpm.goreleaser.com/) on every push to
`master`. The Arch package is adapted for `/usr/lib` so it installs cleanly on
Arch/CachyOS:

```sh
sudo pacman -U libfprint-*.pkg.tar.zst
```

Installing replaces the system `libfprint-2.so`. Then enroll with `fprintd-enroll`
and wire up `pam_fprintd` for login/`sudo` as usual.

## Status

Working end-to-end on real hardware: firmware flash → `enroll` → `verify` →
match. Known rough edge: an `libusb device still referenced` warning at process
exit (an async read-loop cleanup detail; harmless for the CLI examples).

## Credits & license

- Based on [Infinytum/libfprint](https://github.com/Infinytum/libfprint)
  (`driver/538d`), itself a fork of
  [goodix-fp-linux-dev/libfprint](https://github.com/goodix-fp-linux-dev/libfprint)
  and freedesktop [libfprint](https://gitlab.freedesktop.org/libfprint/libfprint).
- Protocol reverse-engineering by the goodix-fp-linux-dev community.
- LGPL-2.1+ (see `COPYING`). Includes NIST NBIS code (see the license notes).
