# Prospector Scanner v2.2.2 Release Notes

**Release Date**: August 2026
**Type**: Patch release (keyboard-side fix only)

## Summary

Fixes two regressions introduced by v2.2.1's burst/silent advertising duty
cycle: uni-body keyboards failed to build, and split-central keyboards could
get the duty cycle stuck on forever, making the keyboard appear only
intermittently on the scanner. Tracked in
[issue #24](https://github.com/t-ogura/zmk-config-prospector/issues/24) and
[issue #22](https://github.com/t-ogura/zmk-config-prospector/issues/22).

The scanner firmware itself is unchanged — only the keyboard-side module needs
to be updated.

## Bug Fixes

- **Uni-body keyboards fail to build on v2.2.1**
  ([issue #24](https://github.com/t-ogura/zmk-config-prospector/issues/24))
  - Symptom: `error: 'CONFIG_PROSPECTOR_SPLIT_PARTIAL_BURST_MS' undeclared`
    when building a non-split keyboard with
    `CONFIG_ZMK_STATUS_ADVERTISEMENT=y`. Split keyboards built fine.
  - Root cause: the burst/silent Kconfig symbols depend on
    `ZMK_SPLIT_ROLE_CENTRAL`, so they don't exist on uni-body builds — but
    one of the three places referencing them in the advertising work handler
    was missing its preprocessor guard.
  - Fix: fallback `#define`s cover non-central builds. The affected branch is
    dead code there (the split-partial state can never be entered), so
    runtime behavior is unchanged.

- **BURST/SILENT cycle locked on forever for split centrals**
  ([issue #22](https://github.com/t-ogura/zmk-config-prospector/issues/22))
  - Symptom: the keyboard appears on the scanner only sporadically — visible
    for ~200 ms out of every 2 s — even though both halves are fully
    connected and typing works. Intermittent across reboots, which made it
    look like a radio or hardware problem.
  - Root cause: the module counts split-peripheral connections via a
    `bt_conn_cb_register()` callback, but that API only reports connections
    established *after* registration. ZMK's split central starts scanning
    earlier in boot than the module's init; when the peripheral pairs within
    that window (typical when both halves power on together), the connection
    is never counted, `prospector_split_fully_connected()` never turns true,
    and the v2.2.1 burst/silent cycle stays engaged permanently.
  - Fix: at init, the module now seeds the peripheral count from
    already-established connections (`bt_conn_foreach`) before registering
    the callback.

## Compatibility

- **Scanner firmware**: No change required. v2.2.0 through v2.2.2 scanner
  builds are functionally identical.
- **Keyboard firmware**: Update the module revision in your keyboard's
  `west.yml`:
  ```yaml
  - name: prospector-zmk-module
    remote: prospector
    revision: v2.2.2
    path: modules/prospector-zmk-module
  ```
- **BLE protocol**: Unchanged. v2.2.2 keyboards remain compatible with older
  scanners and vice versa.
- **Kconfig**: No new options; the v2.2.1 options and defaults are unchanged.

## Migration

Bump the `revision` as above, `west update`, rebuild, and flash the keyboard.
No scanner reflash and no config changes required.

Uni-body keyboards that could not build on v2.2.1 can simply move straight
from v2.2.0 to v2.2.2.

## Verified On

- xiao_ble + hummingbird (uni-body) with
  `CONFIG_ZMK_STATUS_ADVERTISEMENT=y`: reproduces the exact #24 compile
  error on v2.2.1, builds cleanly on v2.2.2.
- aerogu38 (split central, 1 peripheral) on Zephyr 4.1 / ZMK main: builds
  cleanly; connection-seeding path compiled in and linked.

## Credits

- Uni-body build failure report: community user (via
  [#24](https://github.com/t-ogura/zmk-config-prospector/issues/24))
- Root-cause analysis: [@t-ogura](https://github.com/t-ogura)
