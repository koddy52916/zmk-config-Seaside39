# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

ZMK firmware configuration for **Seaside39**, a wireless split keyboard (right-hand trackball) built by converting a Keyball39 to use a Seeed XIAO BLE nRF52840 MCU. This repo contains only firmware *config* (keymap, Kconfig, devicetree overlays, `west.yml` manifest) — the actual ZMK/Zephyr source is pulled in during CI, not stored here.

- R (right) side = ZMK split **central**, has the PMW3610 trackball sensor, connects to hosts (PC/iPad/etc.) over BLE.
- L (left) side = ZMK split **peripheral**, connects only to the R side.

## Build / CI

There is no local Zephyr/ARM toolchain in this repo or dev environment. **All builds happen via GitHub Actions** (`.github/workflows/build.yml`), which calls the reusable `zmkfirmware/zmk/.github/workflows/build-user-config.yml` workflow. The board+shield build matrix is defined in `build.yaml` (produces `Seaside39_R rgbled_adapter`, `Seaside39_L rgbled_adapter`, and `settings_reset` firmware).

Workflow:
1. Edit config files, commit, `git push` to `main`.
2. Check the run with `gh` CLI: `gh run list --limit 1`, then `gh run view <id>` / `gh run view <id> --log-failed` to inspect failures.
3. Download the `.uf2` artifacts from the successful run's Artifacts.
4. Flash: double-tap the XIAO's reset button to enter bootloader mode (shows up as a `XIAO SENSE` USB drive), copy the `.uf2` onto it. Flash `settings_reset-*.uf2` first only when BLE bond/identity-related config changed (e.g. `CONFIG_BT_PRIVACY`), otherwise flash the shield firmware directly.

**Important pitfall (bit the project for ~9 months, Oct 2025–Jul 2026):** `config/west.yml` and `.github/workflows/build.yml` previously tracked `revision: main` for `zmk` and used `build-user-config.yml@main`. Upstream ZMK `main` broke this board's build (Zephyr 4.1 board-qualifier migration), and CI silently failed on every push without anyone noticing (no local build to catch it). **Always confirm a push actually produced a successful CI run (`gh run list`) before assuming a config change took effect on real hardware** — a config change that never built successfully was never flashed.

`zmk`, `zmk-rgbled-widget`, and the build workflow ref are now **pinned to `v0.3.0`** in `config/west.yml` / `build.yml` for this reason. If bumping these versions, pin `zmk` and `zmk-rgbled-widget` to the *same* ZMK release/tag — mismatched versions between the workflow, `zmk`, and community modules (`zmk-rgbled-widget`, `zmk-pmw3610-driver`) cause Kconfig/devicetree-alias errors that are easy to misattribute to unrelated config changes. `zmk-pmw3610-driver` is left on `main` since it hasn't had a new commit since April 2024 (effectively frozen).

## Repo structure

- `config/west.yml` — manifest pulling in `zmk` (core), `zmk-pmw3610-driver` (trackball), `zmk-rgbled-widget` (status LED), `prospector-zmk-module` (currently disabled, see below).
- `config/Seaside39.keymap` — the keymap (see below).
- `config/boards/shields/Seaside39/`:
  - `Seaside39.dtsi` — shared devicetree: physical key layout, matrix transform, kscan (shared by L and R).
  - `Seaside39_R.overlay` / `Seaside39_L.overlay` — per-side devicetree: column GPIOs, and (R only) SPI + PMW3610 trackball node.
  - `Seaside39_R.conf` / `Seaside39_L.conf` — per-side Kconfig. Nearly all BLE/trackball/LED tuning lives in `Seaside39_R.conf` since R is the central/host-facing side.
  - `Kconfig.defconfig` — sets `ZMK_SPLIT`, `ZMK_SPLIT_ROLE_CENTRAL` (R only), and keyboard name per side.
- `build.yaml` — GitHub Actions build matrix.
- `docs/buildguide.md` — end-user hardware assembly + firmware flashing guide (Japanese).

## Keymap structure (`config/Seaside39.keymap`)

Layers: `default_Win` (0), `default_iOS` (1), `MOUSE` (4), `Function` (2), `Control` (3, note: despite being defined last it's `&lt 3`/layer index 3 — check `#define`s at top for `MOUSE`/`SCROLL`/`NUM` layer numbers before assuming layer order from file order).

- Two base layers exist because Windows and iOS/macOS use different modifier conventions (Ctrl vs Cmd, etc.) — `default_iOS` uses `LEFT_GUI`/`LC(SPACE)` where `default_Win` uses `LCTRL`/`GRAVE`.
- BLE profile switching lives on the `Control` layer:
  - Row 1: raw `&bt BT_SEL 0/1/2` — switches BLE profile only.
  - Row 3: `&bt0`/`&bt1`/`&bt2` macros — switch BLE profile **and** toggle the `default_iOS` layer on/off (`bt2` is the iPad-oriented profile: it also turns `default_iOS` on; `bt0`/`bt1` turn it off). Prefer these over the raw row when the target host's modifier layout differs.
- Trackball behavior (`automouse-layer`, `scroll-layers`) is configured on the `trackball@0` node in `Seaside39_R.overlay`, not in the keymap file.

## Known-sensitive Kconfig options (don't re-enable casually)

These are currently commented out in `Seaside39_R.conf` after being root-caused as build- or feature-breaking; if re-enabling, expect to re-verify against a real GitHub Actions build before trusting the result:

- `CONFIG_BT_PRIVACY=y` — conflicts with `BT_SCAN_WITH_IDENTITY` auto-selected by `ZMK_SPLIT_ROLE_CENTRAL` on this ZMK version; causes a hard Kconfig build error.
- `CONFIG_BT_SECURITY_LESC=n` / `CONFIG_BT_SECURITY_FORCE_LESC=n` — forcing legacy (non-LESC) pairing is suspected of breaking iOS pairing.
- `CONFIG_BT_MAX_CONN=2` — broke switching between two already-bonded host profiles (PC/iPad): the split-central link permanently consumes one connection slot, leaving effectively one spare slot, which isn't enough headroom during profile hand-off.

`CONFIG_ZMK_STATUS_ADVERTISEMENT` / `CONFIG_BT=y` / `CONFIG_BT_PERIPHERAL=y` (Prospector scanner support) are also commented out — enabling them previously broke iOS pairing outright.
