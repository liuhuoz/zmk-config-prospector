# Prospector Scanner v2.2.1 Release Notes

**Release Date**: May 2026
**Type**: Patch release (keyboard-side fix only)

## Summary

Fixes a v2.2.x regression where split-central keyboards could fail to bring up
all peripherals at cold boot when the keyboard was running prospector status
advertisement (`CONFIG_ZMK_STATUS_ADVERTISEMENT=y`). Symptoms and root cause
are tracked in [issue #20](https://github.com/t-ogura/zmk-config-prospector/issues/20).

The scanner firmware itself is unchanged — only the keyboard-side module needs
to be updated.

## Bug Fixes

- **Split central peripheral discovery contention**
  ([issue #20](https://github.com/t-ogura/zmk-config-prospector/issues/20))
  - Symptom: Cold boot leaves one peripheral disconnected; recovery requires
    simultaneous reset of all peripherals. With
    `CONFIG_ZMK_STATUS_ADVERTISEMENT=n` the issue disappears.
  - Root cause: Prospector status adv competes with ZMK split scan/connect for
    the single legacy advertising set. The 150 µs `LL_CONNECT_REQ` window
    (BLE T_IFS) is regularly preempted by adv tx, dropping the connect request.
  - Fix: While the central is still bringing up peripherals, prospector now
    runs a burst/silent cycle (200 ms ADV burst + 1.8 s radio silence by
    default) so the central gets uncontested radio time for scan and
    `CONNECT_REQ` tx. Once all expected peripherals are connected, normal adv
    behavior resumes.

## New Kconfig

The new behavior is governed by three Kconfig options on the keyboard side:

| Option | Default | Notes |
|---|---|---|
| `CONFIG_PROSPECTOR_EXPECTED_PERIPHERAL_COUNT` | `1` if split central, else `0` | Number of split peripherals required before the cycle disengages |
| `CONFIG_PROSPECTOR_SPLIT_PARTIAL_BURST_MS` | `200` | Burst window in milliseconds |
| `CONFIG_PROSPECTOR_SPLIT_PARTIAL_SILENT_MS` | `1800` | Silent window in milliseconds (adv set fully released) |

**Multi-peripheral builds** (e.g. central + right half + trackball, or
central + 2 halves) should override the count:

```conf
CONFIG_PROSPECTOR_EXPECTED_PERIPHERAL_COUNT=2
```

**Standalone (non-split) keyboards** are unaffected by default — the cycle
never engages.

**Opt-out**: setting `CONFIG_PROSPECTOR_EXPECTED_PERIPHERAL_COUNT=0` on a split
central keeps the prior v2.2.0 advertising behavior.

## Compatibility

- **Scanner firmware**: No change required. v2.2.0 and v2.2.1 scanner builds
  are functionally identical.
- **Keyboard firmware**: Update the module revision in your keyboard's
  `west.yml`:
  ```yaml
  - name: prospector-zmk-module
    remote: prospector
    revision: v2.2.1
    path: modules/prospector-zmk-module
  ```
- **BLE protocol**: Unchanged. v2.2.1 keyboards remain compatible with v2.2.0
  scanners and vice versa.

## Migration

For existing v2.2.0 users, no config changes are required for typical 2-half
splits — the default `EXPECTED_PERIPHERAL_COUNT=1` covers them. Multi-peripheral
builds should add the override above, then rebuild and flash the keyboard.

## Verified On

- Cornix TB (split central + right half + trackball, 2 peripherals) on
  Zephyr 4.1 / ZMK main snapshot 2026-01-14. Cold boot reliably brings up all
  peripherals; reconnect after disconnect works without manual intervention.

## Credits

- Reproduction and root-cause analysis: [@t-ogura](https://github.com/t-ogura)
- Tracking issue: [#20](https://github.com/t-ogura/zmk-config-prospector/issues/20)
