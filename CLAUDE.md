# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single-purpose fork of libfprint (1.94.1 base, from `Infinytum/libfprint`
`driver/538d`) whose only reason to exist is the **`goodixtls53xd` driver for the
Goodix `27c6:538d`** USB fingerprint sensor. Do not treat this as general
libfprint: CI and the intended build compile only that one driver. Ignore the
other ~30 drivers unless a change specifically requires them.

The 538d **requires community firmware `GF5298_GM168SEC_APP_13016`** flashed onto
the sensor first (via the separate `goodix-fp-dump` tool); the driver rejects any
other firmware version string. The sensor speaks the goodixtls-generation
protocol: a pack layer over USB plus an OpenSSL **TLS-PSK** session with an
all-zero PSK.

## Build / test / format

```sh
meson setup _build -Ddrivers=goodixtls53xd -Dudev_rules=enabled
ninja -C _build
sudo ./_build/examples/{img-capture,enroll,verify}   # hardware tests, need a flashed 538d
scripts/uncrustify.sh                                 # format (CI style)
```

- On GCC 14+ add `-Dc_args=-Wno-error=incompatible-pointer-types` (this is
  1.94.1-era code that predates several warnings becoming default errors).
- `-Ddrivers=goodixtls53xd` avoids pixman/nss/cairo that other drivers pull in.
- The driver's umockdev test infra is not set up here; validate on real hardware.

## Architecture (the goodixtls stack)

`libfprint/drivers/goodixtls/` is layered — a new TLS device is a concrete file
on top of shared transport:
- `goodix_proto.c` — wire framing: `[cmd][u16 len][payload][checksum]`, checksum
  `(0xAA - sum) & 0xFF`, plus the outer "pack" envelope (`flags` 0xA0 message /
  0xB0 TLS). **Received-packet checksums are computed but not enforced** (TODOs
  in `goodix.c`).
- `goodix.c` — USB transport + async command layer (`goodix_send_protocol`,
  `goodix_receive_done`, per-command timeout via `fpi_device_add_timeout`).
- `goodixtls.c` — OpenSSL TLS-PSK server run over a socketpair on a `pthread`;
  the device is the TLS client, image data is tunnelled through it.
- `goodix53xd.c` / `.h` — the concrete `FpImageDevice` for 538d: `id_table` =
  `{0x27c6, 0x538d}`, firmware string check, activate SSM (nop → enable chip →
  fw check → PSK read → reset → OTP → MCU idle → upload config), scan SSM
  (FDT-down → FDT-mode → get image), then image assembly.

Drivers are async state machines (`fpi-ssm.h`) driven by USB transfers
(`fpi-usb-transfer.h`). Image devices deliver a raw frame; NBIS
(`libfprint/nbis/`, MINDTCT + Bozorth3) does minutiae extraction and matching.

## Two fixes that make it work — do NOT regress these

1. **`goodix.c` `goodix_send_nop()` passes `timeout_ms=0`, not `GOODIX_TIMEOUT`.**
   The NOP is fire-and-forget (`reply=FALSE`) and is completed synchronously by
   the `goodix_receive_done()` call right after it. `goodix_send_pack()` runs a
   nested GLib main loop inside `fpi_usb_transfer_submit_sync()`; with a timeout
   armed, that nested loop dispatches it before the cancel, yielding a spurious
   "Command timed out: 0x00" that killed `verify`'s activation. Arming a timeout
   here is both pointless and harmful — the USB transfer keeps its own timeout.

2. **`goodix53xd.c` `scan_on_read_img()` AVERAGES the captured frames and
   NEAREST-NEIGHBOUR upscales (`GOODIX53XD_ENLARGE_FACTOR = 2`) into one stable
   image.** The original swipe assembly (`fpi_do_movement_estimation` +
   `fpi_assemble_frames`) produced a non-deterministic 192×N staircase (the
   `height is -711` log) — different every capture, so enrolled/verified prints
   shared no minutiae (Bozorth 0–3/24). Averaging → stable minutiae (≈33/24).
   **Bilinear upscale was tried and reverted**: it over-smooths this tiny 64px
   sensor, MINDTCT latches onto noise minutiae, matches drop to 0/24. Keep
   nearest-neighbour unless you also add contrast recovery (e.g. background
   subtraction against the calibration frame, which the driver captures into
   `empty_img` but does not yet use).

Sensor is 64×80, `FP_SCAN_TYPE_PRESS`, `bz3_threshold = 24`, EP in `0x83` / out
`0x01`, PSK = SHA-256 of the all-zero PSK, flags `0xbb020001`.

## 27c6:538d bookkeeping

`0x538d` is the only entry in the driver's `id_table`. If a driver ever stops
supporting a PID here, remove it from `whitelist_id_table[]` in
`libfprint/fprint-list-udev-hwdb.c` and regenerate `data/autosuspend.hwdb`
(`ninja -C _build sync-udev-hwdb`) or the hwdb test fails on duplicates.

## CI / packaging (`.github/workflows/main.yml`)

Triggers on push to `master`; builds only `goodixtls53xd`, forces
`-Dudev_rules=enabled` (a single non-`udev`-helper driver leaves rules off by
default, and nFPM needs `70-libfprint-2.rules`), `-Ddoc=false`, then produces
deb/rpm/arch packages via nFPM. `nfpm_{deb,rpm,arch}_sample.yaml` hardcode
`./build/` paths and per-distro lib dirs — the **arch** template was corrected to
`/usr/lib` (not `/usr/lib64`) for Arch/CachyOS, and all three `depends:` trimmed
to the lean runtime set (`glib2`/`libgusb`/`openssl`). `COMMITID`/`LIBVERSION`
placeholders are `sed`-filled by the workflow.

## Known rough edges (safe to leave)

- `libusb device still referenced` at process exit — async read-loop transfer
  isn't drained before `libusb_exit`; cosmetic for the CLI examples.
- Several unused static functions remain in `goodix53xd.c` (unwired calibration
  / OTP-write scaffolding) — pre-existing, not part of the active path.
