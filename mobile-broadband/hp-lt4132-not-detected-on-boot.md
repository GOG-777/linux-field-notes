# HP lt4132 LTE Module Not Detected on Boot (EndeavourOS / Arch Linux)

## The Problem

The HP lt4132 LTE/HSPA+ 4G Module (found in HP EliteBook laptops) appears in
`lsusb` but is never detected by ModemManager after a cold boot. Running
`mmcli -L` returns nothing, and `/dev/ttyUSB*` devices either never appear or
disappear within 10 seconds of boot.

## Symptoms

- `mmcli -L` returns "No modems were found"
- `ls /dev/ttyUSB*` returns "No such file or directory"
- `lsusb` correctly shows: `HP, Inc HP lt4132 LTE/HSPA+ 4G Module`
- Modem works after manually restarting ModemManager multiple times
- `dmesg` shows ttyUSB0-4 appearing at boot then immediately disconnecting

## Root Causes Found (Three Separate Issues)

### 1. usb_modeswitch Breaking the Device

The system usb_modeswitch config for this device at
`/usr/share/usb_modeswitch/03f0:a31d` contained `Configuration=0`, which
sets the USB device to an unconfigured state, stripping all interfaces off it.
Every boot, usb_modeswitch fired and killed the device.

### 2. NetworkManager Interfering on Boot

NetworkManager was managing the modem's CDC Ethernet interface
(`enp0s20f0u3c2`) as a regular ethernet port. This caused the modem to reset
and disconnect all ttyUSB ports within seconds of boot.

### 3. ModemManager Waiting for a Device That Never Appears

A leftover systemd drop-in (`wait-for-modem.conf`) was telling ModemManager
to wait for `/dev/cdc-wdm0` (an MBIM device) before starting. Since this
modem uses the `option` driver instead, that device never appeared, causing
ModemManager to timeout and fail on every boot.

## The Fix

### Step 1: Disable usb_modeswitch for This Device

```bash
sudo rm /usr/share/usb_modeswitch/03f0:a31d
sudo rm /etc/usb_modeswitch.d/03f0:a31d
```

If you want to keep a config file instead of deleting it, make sure
`DisableSwitching=1` is on its own line and not commented out:

```bash
printf "# HP lt4132 - do not switch\nDisableSwitching=1\n" | sudo tee /usr/share/usb_modeswitch/03f0:a31d
```

Protect the fix from being overwritten by package updates:

```bash
sudo mkdir -p /etc/pacman.d/hooks
sudo nano /etc/pacman.d/hooks/fix-lt4132-modeswitch.hook
```

```ini
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = usb_modeswitch

[Action]
Description = Restoring HP lt4132 modeswitch fix...
When = PostTransaction
Exec = /bin/sh -c 'rm -f /usr/share/usb_modeswitch/03f0:a31d'
```

### Step 2: Stop NetworkManager from Touching the Modem Interface

```bash
sudo nano /etc/udev/rules.d/99-nm-unmanaged-modem.rules
```

Add:
```
ACTION=="add", SUBSYSTEM=="net", ATTRS{idVendor}=="03f0", ATTRS{idProduct}=="a31d", ENV{NM_UNMANAGED}="1"
```

```bash
sudo udevadm control --reload-rules
```

### Step 3: Remove the Broken ModemManager Dependency

```bash
sudo rm /etc/systemd/system/ModemManager.service.d/wait-for-modem.conf
sudo systemctl daemon-reload
```

### Step 4: Harden ModemManager

```bash
sudo systemctl edit ModemManager.service
```

Add:
```ini
[Service]
Restart=always
RestartSec=2s
OOMScoreAdjust=-1000
```

### Step 5: Auto-Connect on Boot

Create `/usr/local/bin/cellular-guard.sh`:

```bash
#!/bin/bash
sleep 30

while true; do
    if ! pgrep -x ModemManager > /dev/null; then
        systemctl restart ModemManager
        sleep 15
        continue
    fi

    STATUS=$(mmcli -m any 2>/dev/null | grep "state:" | awk '{print $2}')

    if [ "$STATUS" = "registered" ]; then
        mmcli -m any --simple-connect="apn=glosecure" 2>/dev/null
        sleep 5
    fi

    sleep 15
done
```

```bash
sudo chmod +x /usr/local/bin/cellular-guard.sh
```

Create `/etc/systemd/system/cellular-guard.service`:

```ini
[Unit]
Description=Cellular Connection Guard
After=ModemManager.service

[Service]
ExecStart=/usr/local/bin/cellular-guard.sh
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now cellular-guard.service
```

## Connecting to the Network

NetworkManager's PPP mode does not work with this modem. Connect via
ModemManager directly:

```bash
mmcli -m 0 --simple-connect="apn=YOUR_APN"
```

For Glo Nigeria the APN is `glosecure`.

## Hardware Info

- Device: HP EliteBook 830 G5
- Modem: HP lt4132 (USB ID: 03f0:a31d, underlying chip: Huawei ME906s-158)
- OS: EndeavourOS (Arch Linux)
- Kernel: 7.0.5-arch1-1
- Drivers: option, cdc_ether

