# HP Pavilion Firmware Tools

Linux tools for exploring, monitoring, and controlling HP Pavilion firmware features through ACPI, WMI, and embedded-controller interfaces.

> **Status:** Experimental and hardware-specific.

## Current functionality

The first implemented feature is a battery charge-limit controller using the HP firmware's ACPI/WMI interface. It has been verified on:

- **Model:** HP Pavilion Laptop 15-eh3xxx
- **Motherboard:** 8BC7
- **BIOS:** F.05

### Battery charge control

The controller uses an 80% charge limit and a 75% resume threshold:

```text
Battery >= 80%  → INHIBIT charging
Battery <= 75%  → AUTO charging
Battery 75–80%  → Keep the current state
AC disconnected → No charging-control action
```

It re-applies the policy after AC reconnect and can start automatically through systemd.

## Firmware interface

The firmware exposes battery-control methods through HP WMI/ACPI. DSDT analysis identified these methods:

- `ABES`
- `GBCO`
- `SBCO`

Relevant embedded-controller fields include `MBTS`, `BDSO`, `ADPP`, `SHEN`, `BDVO`, `SBCW`, `SDVO`, and `BATF`. These values were verified experimentally on the supported system:

```text
MBTS = 0x01
BDSO = 0x01
ADPP = 0x01
SHEN = 0x01
```

The tested charging commands and reported states are:

| Command | Mode | GBCO | BDVO |
| --- | --- | --- | --- |
| `SBCO 0 0 0 0` | AUTO | `0x00` | `0x80` |
| `SBCO 0 5 0 0` | INHIBIT | `0x04` | `0xC5` |

The inhibit state was experimentally verified at 80%: capacity was 80, battery status was “Not charging,” `GBCO` reported `0x04`, and `BDVO` was `0xC5`.

## Requirements

The controller uses the Linux `acpi_call` interface. The `acpi_call` kernel module must be available. The script loads it with `modprobe` if `/proc/acpi/call` is not present.

## Installation

Install the controller:

```sh
sudo cp battery/hp-battery-limit /usr/local/bin/hp-battery-limit
sudo chmod 755 /usr/local/bin/hp-battery-limit
```

Install and enable the systemd service:

```sh
sudo cp systemd/hp-battery-limit.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now hp-battery-limit.service
```

Check the service and view its logs:

```sh
systemctl status hp-battery-limit.service
journalctl -u hp-battery-limit.service -f
```

## Hardware compatibility and safety

The firmware mappings documented here were verified only on the HP Pavilion Laptop 15-eh3xxx with motherboard 8BC7 and BIOS F.05. Do not assume the same ACPI methods, embedded-controller fields, or command mappings apply to another model or BIOS version.

Firmware writes can behave differently across hardware. Investigate new models using read-only methods before attempting writes, and enable only hardware-specific commands that have been experimentally verified.

## Reverse engineering

The battery-control interface was identified by examining the system DSDT and tracing the HP WMI/embedded-controller methods used by the firmware:

```text
DSDT / ACPI
    ↓
HP WMI methods
    ↓
EC fields / commands
    ↓
Read-only verification
    ↓
Controlled firmware write
    ↓
Read-back verification
    ↓
Userspace policy
```

The project follows this principle: **Read first, understand second, write third.**

## Roadmap

Possible future work includes:

- HP Pavilion firmware capability discovery
- Additional battery and power controls
- Thermal, fan, and hardware-sensor interfaces
- Performance and platform power controls
- Model-specific capability profiles
- A cleaner userspace interface
- Support for additional HP Pavilion models
