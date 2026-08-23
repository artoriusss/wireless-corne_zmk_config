# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

The ZMK user config for **unicorne** — a Boardsource Wireless Corne SMT (split 3x6+3, nRF52840,
BLE, ZMK Studio capable), forked from Boardsource's official config. There is no application
source here: ZMK itself is fetched by west at build time. This repo supplies only the vendored
board definition, the keymap, and the CI build matrix. Hardware facts (expansion header pinout,
battery dimensions, flashing steps) live in `README.md` — keep that upstream content intact.

### Sibling repo: `/home/mro/Documents/keyboards/qmk-config` (cleo, wired, QMK)

`cleo` (wired, QMK) and `unicorne` (wireless, ZMK) are **kept in sync — they must behave
identically**. The keymap here is a port of `qmk-config/cleo/v2_02/keymaps/keymaps_base.c`.
Only this ZMK repo is in scope for edits, but that file is the reference when a layer's intent
is unclear.

Correspondence:

| concept | QMK (cleo) | ZMK (unicorne) |
| --- | --- | --- |
| layers | `_QWRTY _CLMAK _ROVER _SYMBL _NAVIG _FUNCT _ADJST` | same names/order, `#define QWRTY 0` … `ADJST 6` |
| momentary layer | `MO(_X)` | `&mo X` |
| default layer swap | `DF(_QWRTY)` / `DF(_CLMAK)` | `&to QWRTY` / `&to CLMAK` |
| home-row mods | `LGUI_T(KC_A)` etc. | `&hml LGUI A` / `&hmr …` hold-tap behaviors |
| tap-hold timing | `TAPPING_TERM 200`, `FLOW_TAP_TERM 150` (`cleo/v2_02/config.h`) | `tapping-term-ms = <200>`, `require-prior-idle-ms = <150>` |
| space/hyper | `HYPR_T(KC_SPC)` | `&mt HYPER SPACE` (`HYPER` = `LS(LC(LA(LGUI)))`) |
| blank / passthrough | `XXXXXXX` / `_______` | `&none` / `&trans` |

Known intentional divergence: the `ADJST` layer. QMK puts RGB matrix controls there; ZMK puts
BLE profile management (`&bt BT_SEL 0-4`, `&bt BT_CLR`, `&out OUT_TOG`) there, since those have
no QMK equivalent. Media and brightness keys on that layer do match.

**When changing a keymap, change the matching layer in both repos** (edit only ZMK here; flag
the corresponding QMK edit to the user).

## Layout / build

- `config/blecorne.keymap`, `config/blecorne.conf` — the keymap and Kconfig actually built.
  The filename must match `BOARD` (`blecorne`, set in `Kconfig.defconfig`) — that is how ZMK
  locates the user keymap. A single keymap file covers both halves.
- `boards/boardsource/blecorne/` — vendored Zephyr hardware-model-v2 **board** (not a shield):
  `blecorne.dtsi` (shared: matrix transforms, flash partitions, battery divider, `nice_view_spi`),
  `blecorne_left.dts` / `blecorne_right.dts` (per-half kscan GPIO matrix; right adds
  `col-offset = <6>`), `Kconfig.defconfig` (left is `ZMK_SPLIT_ROLE_CENTRAL`, `ZMK_SPLIT=y`).
  `boards/boardsource/blecorne/blecorne.keymap` is the vendor's stock default keymap — it is NOT
  what gets built; ignore it unless deliberately changing the board's upstream default.
- `zephyr/module.yml` sets `board_root: .`, which is what makes the vendored board visible to
  the ZMK build.
- `build.yaml` — the GitHub Actions matrix: `blecorne_left` (with `studio-rpc-usb-uart` snippet
  and `-DCONFIG_ZMK_STUDIO=y`), `blecorne_right`, plus both halves with the `settings_reset`
  shield. `shield: nice_view` lines are commented out; enabling one also requires
  `CONFIG_ZMK_DISPLAY=y` in `config/blecorne.conf`.
- Both `config/west.yml` (`revision: main`) and `.github/workflows/build.yml`
  (`build-user-config.yml@main`) track ZMK **main**, not a release tag. Upstream breakage lands
  here without warning — when a build suddenly fails with no local change, suspect ZMK main
  first, and pin both to a tag (e.g. `v0.3`) together if pinning is wanted.

### Building

Firmware is built in CI on every push/PR — there is no local west workspace and no local build
setup in this repo. Do not assume you can compile; verify keymap changes by reading, and let CI
build. Download the `firmware` artifact from the Actions run; it contains one `.uf2` per matrix
entry.

Flashing: connect a half over USB, double-tap reset to mount the bootloader drive, copy the
matching `.uf2`. When BLE pairing or split peering breaks, flash the `settings_reset` uf2 to
both halves first, then reflash the normal firmware.

## Keymap editing notes

- Physical key positions used by `hold-trigger-key-positions` are documented in the header
  comment of `config/blecorne.keymap` (`KEYS_L`, `KEYS_R`, `THUMBS` for the 3x6+3 layout).
  `hml` restricts to right-hand + thumb keys and `hmr` to left-hand + thumbs; both use
  `flavor = "balanced"` with `hold-trigger-on-release`.
- Every layer must list exactly 42 bindings in row-major order (12/12/12/6).
- The ASCII art above each layer is hand-maintained and already drifts from the bindings in
  places — trust the `bindings` array, and fix the comment when you touch a layer.
