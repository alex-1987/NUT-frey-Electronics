
# Zircon UPS (ID 0001:0000) NUT Configuration Guide

This repository provides a working configuration for **Zircon UPS** devices (and other rebadged units) that identify as `ID 0001:0000 Fry's Electronics` via USB. These devices are often tricky because they do not work with the standard `usbhid-ups` driver.

## 1. Prerequisites

Ensure you have NUT installed on your Debian/Ubuntu/Proxmox system:

```bash
sudo apt update && sudo apt install nut

```

Verify your UPS is connected and detected:

```bash
lsusb
Bus 003 Device 003: ID 0001:0000 Fry's Electronics
```

---

## 2. Configuration Files

### `/etc/nut/nut.conf`

Set the mode to standalone to allow the local system to manage the UPS.

```ini
MODE=standalone

```

### `/etc/nut/ups.conf`

This is the most important part. The Zircon UPS requires the `nutdrv_qx` driver and `hunnox` subdriver.

```ini
maxretry = 3

[zircon]
    driver = "nutdrv_qx"
    subdriver = "hunnox"
    protocol = q1
    port = "auto"
    vendorid = "0001"
    productid = "0000"
    norating
    novendor
    langid_fix = "0x0409"
    desc = "Zircon UPS"
    ignorelb
    override.battery.charge.low = 20
    default.battery.voltage.high = "13.8"
    default.battery.voltage.low = "11"
    runtimecal = 300,100,600,50

```

---

## 3. Applying Changes

Whenever you modify `nut.conf` or `ups.conf`, you must restart the driver and the services for the changes to take effect.

1. **Restart the Driver:**
```bash
sudo upsdrvctl stop
sudo upsdrvctl start

```


2. **Restart the Services:**
```bash
sudo systemctl restart nut-server nut-client

```



---

## 4. Verifying the Connection

To verify that your system is successfully communicating with the UPS and to see real-time data (like battery charge and voltage), use the `upsc` command followed by the name you defined in brackets in `ups.conf`:

```bash
upsc zircon

```

**Note:** If you named your UPS something else in `ups.conf` (e.g., `[myups]`), you would run `upsc myups`.

---

## 5. Discord Notifications (Optional)

If you want to receive alerts via Discord when the power status changes, you can use the following script.

**Create `/etc/nut/upsalert.sh`:**

```bash
#!/bin/bash
WEBHOOK_URL="YOUR_WEBHOOK_HERE"
case "$1" in
    ONLINE)   MSG="✅ Power Restored!" ;;
    ONBATT)   MSG="⚠️ Power Outage! Running on Battery." ;;
    LOWBATT)  MSG="🚨 Battery Critical!" ;;
    *)        MSG="UPS Event: $1" ;;
esac
curl -H "Content-Type: application/json" -X POST -d "{\"content\": \"$MSG\"}" $WEBHOOK_URL

```

*Remember to `chmod +x /etc/nut/upsalert.sh`.*

---

**Would you like me to add a section on how to troubleshoot common "Data Stale" errors for this specific chipset?**
