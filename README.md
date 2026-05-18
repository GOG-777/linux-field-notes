<div align="center">

<img src="https://endeavouros.com/wp-content/uploads/2021/03/endeavouros-icon.png" width="80" />

# Linux Field Notes

**Real-world Linux troubleshooting documentation.**
Problems encountered, root causes identified, and exact fixes that worked.

[![EndeavourOS](https://img.shields.io/badge/EndeavourOS-7B2FBE?style=for-the-badge&logo=archlinux&logoColor=white)](https://endeavouros.com)
[![KDE Plasma](https://img.shields.io/badge/KDE%20Plasma%206-1D99F3?style=for-the-badge&logo=kde&logoColor=white)](https://kde.org)
[![Wayland](https://img.shields.io/badge/Wayland-FFBC00?style=for-the-badge&logo=wayland&logoColor=black)](https://wayland.freedesktop.org)
[![Arch Linux](https://img.shields.io/badge/Arch%20Linux-1793D1?style=for-the-badge&logo=archlinux&logoColor=white)](https://archlinux.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](LICENSE)

</div>

---

## What This Is

Linux is a fix-it-yourself ecosystem. When something breaks, the fix is out there, but it is rarely documented in one place with a clear explanation of why it works.

This repository exists to change that. Every document here covers a real problem encountered on a real machine. Each entry includes the exact symptoms, the actual root cause (not just a workaround), and the precise commands used to resolve it.

No guesswork. No copy-pasted commands without explanation. No hallucinated fixes.

---

## System Reference

All fixes in this repository were developed and tested on the following setup. Results may vary on different hardware or configurations, but the diagnostic approach applies broadly.

| Component | Details |
|-----------|---------|
| OS | EndeavourOS (rolling, Arch-based) |
| Desktop | KDE Plasma 6.6.5 |
| Session | Wayland |
| Hardware | HP EliteBook 830 G5 |
| CPU | Intel Core i5-8350U (Kaby Lake-U) |
| GPU | Intel UHD Graphics 620 |
| Bootloader | systemd-boot |

---

## Documentation

### Mobile Broadband

| Document | Description |
|----------|-------------|
| [HP lt4132 LTE Module Not Detected on Boot](mobile-broadband/hp-lt4132-not-detected-on-boot.md) | Covers three separate root causes that prevent the HP lt4132 modem from being detected by ModemManager on cold boot, including a usb_modeswitch misconfiguration, NetworkManager interference, and a broken systemd dependency. |

### KDE Plasma

| Document | Description |
|----------|-------------|
| [Plasma Wayland Black Screen on Intel GPU](kde-plasma/wayland-black-screen-intel.md) | Coming soon |

---

## How Each Document Is Structured

Every entry in this repository follows the same structure so that the information is easy to scan and act on.

- **The Problem** - A plain description of what went wrong
- **Symptoms** - Exact error messages, commands that failed, and observable behavior
- **Root Cause** - The actual reason it broke, not just the surface-level fix
- **The Fix** - Step-by-step commands with explanations for what each one does
- **Hardware and Environment** - The specific setup the fix was tested on

---

## A Note on Documentation Quality

Commands in this repository have been executed on a live system and confirmed to work. Each fix explains the reasoning behind it because understanding why a fix works is what allows you to adapt it when your setup differs slightly.

If a fix did not work in a specific case or caused a regression, that is documented too.

---

## Contributing

If you have encountered a similar issue and found a different or better approach, pull requests are welcome. The goal is accuracy, not completeness for its own sake.

---

## Platforms

This repository is mirrored on both GitHub and GitLab.

- GitHub: [github.com/GOG-777/linux-field-notes](https://github.com/GOG-777/linux-field-notes)
- GitLab: [gitlab.com/GOG-777/linux-field-notes](https://gitlab.com/GOG-777/linux-field-notes)

---

<div align="center">

Built from actual pain. Documented so you don't have to feel it too.

</div>
