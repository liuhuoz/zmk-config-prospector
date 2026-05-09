# Prospector Scanner v2.2.1

**🛡️ Patch release: split central peripheral discovery fix**

---

## What's Fixed

**Split central peripheral discovery contention** ([#20](https://github.com/t-ogura/zmk-config-prospector/issues/20))

On v2.2.x split-central keyboards, prospector's status advertisement could
preempt the central's `LL_CONNECT_REQ` (150 µs BLE T_IFS window), causing one
peripheral to silently fail to connect at cold boot. Recovery required
simultaneous reset of all peripherals.

v2.2.1 adds a burst/silent cycle (200 ms ADV burst + 1.8 s radio silence by
default) that runs while the central is still bringing up peripherals,
yielding uncontested radio time for split scan/`CONNECT_REQ`. Once all
expected peripherals are connected, normal adv behavior resumes.

## Multi-peripheral builds

For builds with more than one peripheral (e.g. central + right half +
trackball), set in your keyboard config:

```conf
CONFIG_PROSPECTOR_EXPECTED_PERIPHERAL_COUNT=2
```

Standalone keyboards and typical 2-half splits work with defaults.

## Scope

- **Keyboard module**: updated (this release)
- **Scanner firmware**: no functional change
- **BLE protocol**: unchanged — v2.2.1 keyboards interop with v2.2.0 scanners
  and vice versa

## Migration

Update `west.yml` on the keyboard side:

```yaml
- name: prospector-zmk-module
  remote: prospector
  revision: v2.2.1
  path: modules/prospector-zmk-module
```

Then `west update`, rebuild, and flash. No scanner reflash required.

## Verified

Cornix TB (central + right half + trackball, 2 peripherals) on Zephyr 4.1 /
ZMK main snapshot 2026-01-14. Cold boot reliably brings up all peripherals.

---

**🏷️ Version**: v2.2.1
**📅 Released**: May 2026
**🛡️ Status**: STABLE
**📜 License**: MIT
