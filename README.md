
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

To receive real-time alerts on Discord, you need two things: the notification script and the configuration to trigger it.

### Step A. User Authentication (`upsd.users`)

We define two separate users:

1. **admin**: For administrative tasks and manual commands.
2. **monuser**: A restricted user used only by the `upsmon` service to monitor the status.

**File:** `/etc/nut/upsd.users`

```ini
[admin]
    password = admin_password_here
    actions = SET
    instcmds = ALL
    upsmon primary

[monuser]
    password = monitor_password_here
    upsmon primary

```

### Step B: Create the script

Create a file at `/etc/nut/upsalert.sh`:

```bash
#!/bin/bash
WEBHOOK_URL="YOUR_DISCORD_WEBHOOK_URL"

case "$1" in
    ONLINE)   MSG="✅ Power Restored! Server running on AC." ;;
    ONBATT)   MSG="⚠️ Power Outage! UPS taking over." ;;
    LOWBATT)  MSG="🚨 Battery critically low!" ;;
    SHUTDOWN) MSG="🛑 System shutdown initiated." ;;
    *)        MSG="UPS Event: $1" ;;
esac

curl -H "Content-Type: application/json" -X POST -d "{\"content\": \"**[NUT-Alarm]** $MSG\"}" $WEBHOOK_URL

```

**Important:** Make the script executable:

```bash
chmod +x /etc/nut/upsalert.sh

```

### Step C: Edit `/etc/nut/upsmon.conf`

You must tell `upsmon` to use the script as a `NOTIFYCMD` and define which events should trigger it using `EXEC`.

Add or modify these lines in `/etc/nut/upsmon.conf`:

```ini
# Path to your script
NOTIFYCMD /etc/nut/upsalert.sh

# Monitor line (ensure 'admin' and 'PASSWORD' match your upsd.users)
MONITOR zircon@localhost 1 monuser monitor_password_here primary


# Define which events trigger the script (EXEC)
NOTIFYFLAG ONLINE   SYSLOG+WALL+EXEC
NOTIFYFLAG ONBATT   SYSLOG+WALL+EXEC
NOTIFYFLAG LOWBATT  SYSLOG+WALL+EXEC
NOTIFYFLAG SHUTDOWN SYSLOG+WALL+EXEC

```

---

## 6. Final Step: Refreshing the system

After editing `upsmon.conf`, restart the client to start monitoring:

```bash
sudo systemctl restart nut-client

```

To verify your configuration and see your UPS data, run:

```bash
upsc zircon

```

