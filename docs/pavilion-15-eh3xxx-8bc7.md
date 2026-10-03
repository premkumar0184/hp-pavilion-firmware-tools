# HP Pavilion 15-eh3xxx / 8BC7

## Hardware

Tested system:

```text
Model: HP Pavilion Laptop 15-eh3xxx
Board: 8BC7
BIOS:  F.05
```

## Goal

The investigation sought to determine whether this laptop exposes a firmware-level charging-control interface that can implement a Linux battery charge limit without directly modifying arbitrary embedded-controller registers.

## DSDT findings

The system DSDT exposes the HP WMI device and several battery-related methods:

- `ABES`
- `GBCO`
- `SBCO`

`ABES` returns the embedded-controller field `SHEN`, which corresponds to the firmware's Adaptive Battery Extender status.

The DSDT's battery-control path used by `GBCO` checks `MBTS`, `BDSO`, and `ADPP` before accessing the battery-control state. Other relevant embedded-controller fields include `SHEN`, `BDVO`, `SDVO`, `SBCW`, and `BATF`. The DSDT defines `SHEN` as a one-bit field at EC offset `0xC5`.

## Read-only observations

The following values were read from the tested system:

```text
MBTS = 0x01
BDSO = 0x01
ADPP = 0x01

STOE = 0x00
SHPM = 0x00
BATF = 0x00

SHEN = 0x01
BDVO = 0x80
SBCW = 0x00
SDVO = 0x00
```

`SHEN = 0x01` was independently confirmed through the firmware's `ABES` method.

## Charging control

The firmware's `SBCO` method was tested through the Linux `acpi_call` interface.

### Invalid mode

The command `SBCO 0 8 0 0` returned:

```text
[0x0, 0x0, {0x03, 0x00, 0x00, 0x00}]
```

This indicated that the firmware received and rejected the invalid mode rather than accepting an unknown command.

### AUTO

The command `SBCO 0 0 0 0` was accepted and returned:

```text
[0x0, 0x0, {0x00, 0x00, 0x00, 0x00}]
```

Afterward, the firmware continued to report its normal charging state.

### INHIBIT

The command `SBCO 0 5 0 0` was accepted and returned:

```text
[0x0, 0x0, {0x00, 0x00, 0x00, 0x00}]
```

The firmware then reported:

```text
GBCO = [0x0, 0x4, {0x04, 0xff, 0x00, 0x00}]
BDVO = 0xc5
```

The DSDT's `GBCO` logic recognizes `BDVO = 0xC5` and translates it into status `0x04`.

## Experimental verification

The battery was discharged to 67% and then charged with the controller active. Charging resumed at 67% and was inhibited at 80%.

At 80%, the system reported:

```text
capacity = 80
status   = Not charging
```

At the same time, the firmware reported:

```text
GBCO = [0x0, 0x4, {0x04, 0xff, 0x00, 0x00}]
BDVO = 0xc5
```

This verifies the 80% policy on the tested system.

## AC power behavior

The firmware does not preserve the inhibit state across AC removal. With AC removed, observations were:

```text
AC     = 0
status = Discharging
GBCO   = ... {0x02, ...}
BDVO   = 0x80
```

After AC was connected again, the firmware returned to its normal state:

```text
AC   = 1
GBCO = ... {0x00, ...}
BDVO = 0x80
```

The userspace controller therefore re-evaluates the battery policy when AC state changes and re-applies inhibit when the battery is at or above the configured target.

## Controller policy

The controller uses `TARGET = 80` and `RESUME = 75`:

```text
Battery >= 80%  → INHIBIT
Battery <= 75%  → AUTO
Battery 75–80%  → Keep the current policy
```

This hysteresis prevents repeated mode switching around the target level.

## Boot persistence

The controller runs as the `hp-battery-limit.service` systemd service, which was enabled and verified to start successfully after reboot.

After reboot with AC connected, the controller restored the firmware inhibit state. The system reported:

```text
service  = active
capacity = 100
GBCO     = [0x0, 0x4, {0x04, 0xff, 0x00, 0x00}]
BDVO     = 0xc5
```

## Safety and compatibility

The firmware mappings documented here were verified only on the HP Pavilion Laptop 15-eh3xxx with board 8BC7 and BIOS F.05. They should not be assumed to apply to other HP Pavilion models or BIOS versions.

The project uses the firmware's own ACPI/WMI charging method rather than directly writing an arbitrary EC register. The investigation followed this sequence:

```text
Read
 ↓
Understand
 ↓
Controlled write
 ↓
Read back
 ↓
Observe physical behavior
```

Additional firmware features should be treated as unsupported until independently verified.
