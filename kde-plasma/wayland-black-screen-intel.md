# KDE Plasma Wayland Black Screen on Intel GPU (EndeavourOS / Arch Linux)

## The Problem

After logging into a KDE Plasma Wayland session on a machine with an Intel
integrated GPU, the screen goes completely black immediately after the
"Plasma powered by KDE" splash screen. The system is running but nothing
renders. Switching to Plasma X11 works fine.

---

## Symptoms

- Plasma Wayland session shows splash screen then goes black
- Cursor may or may not be visible on the black screen
- Plasma X11 session loads and works correctly
- XFCE or other lightweight desktop environments load fine on Wayland
- `Ctrl+Alt+F2` does not respond during the black screen
- `nomodeset` kernel parameter allows boot but with degraded display quality
- The issue occurs on cold boot but Wayland may work after logging into X11
  first and then switching sessions

---

## Root Causes Found (Three Separate Issues)

This problem had multiple independent causes that needed to be resolved in
sequence.

### 1. Corrupted KWin Configuration

The KWin compositor configuration file at `~/.config/kwinrc` had become
corrupted. On Wayland, KWin initializes the compositor differently from X11,
and a corrupted config causes it to crash silently before rendering anything.

### 2. Intel Panel Self Refresh (PSR) Bug

Intel integrated GPUs use a power-saving feature called Panel Self Refresh
(PSR). On Kaby Lake and nearby generations, a known kernel bug causes PSR to
produce a completely black screen when KWin Wayland tries to initialize the
display. This is a well-documented issue in the Linux kernel that affects
many Intel laptop users.

### 3. Wrong DRM Device Selected on Cold Boot

On cold boot, the kernel initializes `simpledrm` (a basic framebuffer driver)
before the real `i915` Intel driver takes over. KWin Wayland sometimes grabs
the `simpledrm` device instead of the real Intel GPU during this transition,
resulting in a black screen. After X11 has run once and the drivers have
settled, the correct device is available and Wayland works.

---

## The Fix

Apply all three fixes. Each one addresses a separate layer of the problem.

### Step 1: Reset the KWin Configuration

Back up and remove the corrupted KWin config so it regenerates cleanly on
next login.

```bash
mv ~/.config/kwinrc ~/.config/kwinrc.bak
```

Test if Plasma Wayland loads after this step before continuing. If it does,
the corrupted config was the only issue. If it still black screens, continue
to the next steps.

### Step 2: Disable Intel PSR via Kernel Parameter

This fix needs to be applied to the systemd-boot entry since EndeavourOS uses
systemd-boot by default on UEFI systems.

Find your boot entry file:

```bash
sudo ls /efi/loader/entries/
```

Open the entry file. The filename contains your machine ID and kernel version:

```bash
sudo nano /efi/loader/entries/YOUR_ENTRY_FILE.conf
```

Find the `options` line and add `i915.enable_psr=0` at the end:

```
options root=UUID=xxxx rw quiet splash i915.enable_psr=0
```

To make this survive kernel updates, also add it to the persistent kernel
command line file:

```bash
echo "i915.enable_psr=0" | sudo tee -a /etc/kernel/cmdline
```

> **Note:** If your system uses GRUB instead of systemd-boot, edit
> `/etc/default/grub` and add `i915.enable_psr=0` to
> `GRUB_CMDLINE_LINUX_DEFAULT`, then run `sudo grub-mkconfig -o /boot/grub/grub.cfg`.

### Step 3: Force KWin to Use the Correct DRM Device

Tell KWin Wayland explicitly which GPU device to use so it does not
accidentally grab the simpledrm framebuffer device on cold boot.

First, check which device is your real Intel GPU:

```bash
ls /dev/dri/
```

You are looking for `card1` (or whichever card is NOT simpledrm). Then add
the following to `/etc/environment`:

```bash
sudo nano /etc/environment
```

Add this line:

```
KWIN_DRM_DEVICES=/dev/dri/card1
```

Save the file and reboot. Select Plasma Wayland directly at the SDDM login
screen to confirm the cold boot fix is working.

---

## Bonus: Fixing the SDDM Login Screen Default Session

If SDDM defaults to XFCE or another session instead of Plasma Wayland after
a cold boot, create a default session config:

```bash
sudo mkdir -p /etc/sddm.conf.d
sudo nano /etc/sddm.conf.d/default-session.conf
```

Add:

```ini
[General]
DefaultSession=plasma.desktop
```

Also check what session SDDM has saved as the last used session:

```bash
sudo cat /var/lib/sddm/state.conf
```

If it shows a session other than `plasma.desktop`, edit the file and update
the `Session` value.

---

## Bonus: Cleaning Up a Partial GDM Install

If you attempted to switch from SDDM to GDM at any point and it left the
system in a broken state, clean it up with:

```bash
# Remove GDM if it was installed
sudo pacman -Rns gdm

# Re-enable SDDM
sudo systemctl enable sddm
sudo systemctl disable gdm

# Remove leftover GNOME session files that clutter the session picker
sudo rm -f /usr/share/wayland-sessions/gnome.desktop
sudo rm -f /usr/share/wayland-sessions/gnome-classic.desktop
```

---

## Verification

After applying all fixes and rebooting, confirm Wayland is running:

```bash
echo $XDG_SESSION_TYPE
```

This should output `wayland`. You can also check KWin is using the correct
device:

```bash
cat /etc/environment | grep KWIN
```

---

## Hardware and Environment

| Component | Details |
|-----------|---------|
| Device | HP EliteBook 830 G5 |
| GPU | Intel UHD Graphics 620 (Kaby Lake-U, i915 driver) |
| OS | EndeavourOS (Arch Linux) |
| Desktop | KDE Plasma 6.6.5 |
| Session | Wayland |
| Bootloader | systemd-boot |
| Kernel | 7.0.5-arch1-1 |

---

## Related Issues

- Intel PSR black screen is a known upstream kernel bug affecting multiple
  Kaby Lake, Skylake, and Broadwell generation laptops
- The simpledrm vs i915 race condition affects any Intel laptop using
  systemd-boot with a UEFI firmware that initializes a basic framebuffer
  before handing off to the kernel GPU driver
