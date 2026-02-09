# TROUBLESHOOTING.md

## Troubleshooting

### Before you start

**autostream** is designed to be reached over your home network at:

* `http://<hostname>.local/` (example: `http://autostream.local/`)

If `.local` names don’t work on your network/device, try the device IP address instead (see **Advanced → Finding the IP**).

---

### Same Wi-Fi network

For discovery/control to work reliably, your phone/tablet, the **autostream** Raspberry Pi, and your AirPlay speakers must be on the **same LAN/Wi-Fi**.

Check these common issues:

* **Guest Wi-Fi / “Guest network”**: often blocks device-to-device traffic (client isolation).
* **“AP isolation” / “Client isolation”** enabled on your router/AP: prevents devices from talking to each other.
* **VLANs / multiple SSIDs**: your phone might be on one network while speakers are on another.
* **Extenders / mesh nodes** with isolation settings: can block multicast/broadcast.

What to try next:

1. Put your phone/tablet on your **main** Wi-Fi (not guest).
2. Confirm your AirPlay speaker is also on that same Wi-Fi.
3. Try loading:

   ```text
   http://autostream.local/
   ```
4. If it still doesn’t load, try from another device (a laptop on the same Wi-Fi) to rule out a phone/browser issue.

---

### Forgotten PIN

If you’ve forgotten your PIN, you can reset it by editing the file stored on the device’s micro-SD card.

1. Power down the Raspberry Pi, remove the micro-SD card, and insert it into a Windows PC (using an SD adapter if needed).
2. Open the boot/config partition that appears in File Explorer, locate PIN.TXT, and update its contents to your new PIN (overwrite the existing value).
3. Save the file and safely eject the micro-SD card.
4. Reinsert it into the Raspberry Pi, and power it back on.

Once **autostream** starts, you should be able to authenticate using the updated PIN.

---

### Raspberry Pi power problems

**autostream** includes a Raspberry Pi power health check (it reads the Pi’s “throttled/undervoltage” status). Power issues can look like software/network bugs.

Symptoms that strongly suggest under-power:

* Random reboots
* Wi-Fi dropouts / network disappearing
* Audio glitches or “stuttering”
* SD card corruption / sudden boot failures after a power event

What to try next:

* Use a known-good Raspberry Pi PSU (or an equivalent, high-quality supply for your specific Pi model. Note that 'an expensive USB adapter' it not necessarily suitable, especially if you are using a Pi4 or Pi5).
* Avoid powering from a TV USB port or low-power hub.
* If you can reach the UI, check for any **power warning banner** (**autostream**’s code exposes messages like “Under-voltage detected…” via `core/autostream_rpi.py`).

---

### Multiple devices

If you have more than one **autostream** on the same network, you should give each device a **unique hostname**.

Why this matters:

* The default access pattern is `http://<hostname>.local/`.
* Duplicate hostnames can cause “sometimes it loads, sometimes it doesn’t”, or you might reach the wrong unit.

#### To Change the Device Name (hostname) 

Go into the Setup page and enter a new hostname. This can also be set during the intial system setup. Either way, you'll need the PIN.

Choose a unique hostname for each unit, e.g.:

   * `autostream-lounge`
   * `cd-player`
   * `lounge-hifi`

After changing it, confirm the active name:

* From the UI URL you use:

  ```text
  http://<new-hostname>.local/
  ```
* Or via SSH (if you have it enabled) (see **Advanced → SSH/login**):

  ```bash
  hostname
  cat /etc/hostname
  ```

> Note: **autostream** also makes a best-effort attempt to refresh mDNS announcements for hostname changes. Depending on your network, it may still take a short while for `.local` discovery to update.


---

### Changed Wi-Fi SSID or password

**autostream** stores the “configured SSID” marker at:

* `/opt/autostream/ssid` (see `core/autostream_wifi_watcher.py`)

If your Wi-Fi name or password changes, the device may no longer be able to connect. **autostream**’s Wi-Fi watcher will attempt recovery by entering **AP mode** under specific conditions.

#### What AP / hotspot mode looks like

![autostream hotspot setup](autostream-hotspot.png)

**autostream**’s AP mode:

* Starts if Wi-Fi is **unconfigured** or still **offline after ~60 seconds** after boot
* Runs for **up to 15 minutes**
* Is **suppressed if wired Ethernet is connected**
* Uses an SSID derived from the Wi-Fi MAC address:
  * `autostream_XXXX` (last 4 hex digits), or fallback `autostream_SETUP`
* Uses a local AP IP of:
  * `192.168.4.1/24`

#### Recovery steps (re-provision Wi-Fi)

1. **Reboot** the **autostream** device.
2. Wait ~2 minutes.
3. On your phone/tablet, look for a Wi-Fi network named like:

   * `autostream_3A7F` (example)
4. Connect to that hotspot.
5. The **autostream** WiFi setup wizard should open automatically. If it doesn't, disconnect
and 'forget' the network, and try again. As last resource, navigate using Safari to:
     ```text
     http://autostream.local/setup
     ```
6. Select your home Wi-Fi SSID from the list and enter it's password.
7. Wait for the device to rejoin your home Wi-Fi, then reconnect your phone back to your home Wi-Fi and load via Safari:

   ```text
   http://<hostname>.local/
   ```

If you miss the AP window:

* AP mode is only available **once per boot** for ~15 minutes.
* Power-cycle/reboot and try again.

---

### If nothing works (quick checklist)

Run through this in order:

1. **Reboot order**

   * Reboot **autostream**
   * Reboot your Wi-Fi router/AP (only if you suspect router issues)
   * Power-cycle the AirPlay speaker(s)
2. **Same network**

   * Phone/tablet + **autostream** + speakers on the same SSID (not guest)
3. **Try both URLs**

   * `http://autostream.local/`
   * `http://<device-ip>/` (see Advanced)
4. **Power**

   * Swap to a known-good Raspberry Pi PSU
5. **Service health**

   * If you can SSH, check `autostream.service` (see Advanced)

---

### "Web interface request denied: No password set in the config"

If you see repeated errors in the Owntone logs like:
```
httpd: Web interface request to '/api/outputs' denied: No password set in the config
```

This means Owntone is blocking API access because it's not configured to allow connections from autostream.

**What causes this:**

Owntone requires EITHER an admin password OR trusted networks to be configured for security. The default configuration has neither, which prevents autostream from controlling outputs and speakers.

**Quick fix (automatic):**

The latest version of autostream automatically fixes this on startup. Simply reboot the device:

```bash
sudo reboot
```

After reboot, autostream's health check will detect the missing configuration and automatically add:
```
trusted_networks = 127.0.0.1, ::1
```

This allows local (localhost) connections without a password, which is safe since autostream runs on the same machine.

**Manual fix (if automatic fix doesn't work):**

1. Edit the Owntone configuration file:
   ```bash
   sudo nano /opt/autostream/owntone/owntone.conf
   ```

2. Add this line anywhere in the file (or uncomment if it exists):
   ```
   trusted_networks = 127.0.0.1, ::1
   ```

3. Save and exit (Ctrl+X, then Y, then Enter)

4. Restart Owntone:
   ```bash
   sudo systemctl restart owntone
   ```

5. Check if it's working:
   ```bash
   journalctl -u owntone --no-pager -n 50 | grep -i "denied\|password"
   ```

   You should no longer see "denied" messages.

**Alternative: Set an admin password (not recommended for autostream):**

Instead of trusted_networks, you could set an admin password in owntone.conf:
```
admin_password = your_password_here
```

However, this would require modifying autostream's code to authenticate with the password, so using `trusted_networks` is the simpler solution.

**Related issues this fixes:**

- Cannot control speaker volume
- Speakers don't appear in autostream UI
- AirPlay streaming fails with "500 Internal Server Error"
- autostream web interface shows "Could not reach Owntone" errors

---

### "Could not reach Owntone" Error

If you see an error message like "Could not reach Owntone at http://localhost:3689", this means **autostream** cannot communicate with the Owntone service that handles AirPlay streaming.

**Common causes:**

1. **Owntone service not running**
   * The Owntone systemd service may have crashed or failed to start
2. **System just rebooted**
   * Owntone may still be starting up (can take 10-30 seconds)
3. **Configuration issue**
   * Owntone configuration file may have syntax errors

**Quick fixes to try:**

1. **Wait 30 seconds** - Owntone may still be starting up after a reboot
2. **Reboot the device** - This will restart all services cleanly
3. **Check service status via SSH** (see below)

**If you have SSH access:**

Check if Owntone is running:

```bash
systemctl status owntone
```

If it shows as "inactive" or "failed", try restarting it:

```bash
sudo systemctl restart owntone
```

Check Owntone logs for errors:

```bash
journalctl -u owntone --no-pager -n 100
```

Common issues in logs:
* **Configuration errors** - Look for "config" or "parse" errors
* **Permission issues** - Look for "permission denied" errors
* **Port conflicts** - Look for "bind" or "address already in use" errors

Verify Owntone is listening on port 3689:

```bash
sudo netstat -tlnp | grep 3689
```

You should see output like:
```
tcp        0      0 0.0.0.0:3689            0.0.0.0:*               LISTEN      12345/owntone
```

If Owntone won't start, check the configuration file:

```bash
sudo nano /opt/autostream/owntone/owntone.conf
```

Look for syntax errors (missing braces, quotes, etc.)

**Still stuck?**

Download logs from the **autostream** UI (Logs page) and check for additional error messages. The full error message from **autostream** now includes diagnostic information to help identify the specific issue.

---

## Advanced

### Downloading logs

There are two ways to access the autostream logs:

1. **Offline “problem” page → Download Logs (ZIP)**

   * If **autostream**’s main UI is down, the nginx “offline” page includes a **Download Logs** button that hits:

     * `/offline/download-logs`
   * This runs `assets/cgi/download-logs.cgi` (via nginx + `fcgiwrap`) and returns a ZIP created from:

     * `/var/log/autostream/*.log`

2. **Logs page → Download (single log file)**

   * The main UI has a Logs page that can download the configured log file via:

     * `/api/log_file`
   * The file path comes from config (`general.log_file`) parsed by `core/autostream_config.py`.

Browser note (important):

* The “Download Log Bundle” is accessible from Safari or a PC - it will not show in “standalone/PWA” mode.
* If downloads don’t work in your current browser, try a different one. **Safari on macOS** or a **Windows browser** is a known workaround.

Where logs live on disk:

* `/var/log/autostream/autostream.log` (common main log file path in this repo)
* `/var/log/autostream/wifi_setup.log` (Wi-Fi/AP mode state machine)
* `/var/log/autostream/update.log` (updater)
* `/var/log/autostream/autostream_rebooter.log` (rebooter service)

---

### SSH/login and service checks

This repo includes systemd units intended to run from an install location like `/opt/autostream` (see `system/systemd/*.service`). Whether SSH is enabled depends on your OS image/install (not defined in this repo), but if you can SSH in, these checks are the fastest way to diagnose issues.

#### Find the IP (if `.local` doesn’t work)

On your router:

* Look for a DHCP client named `autostream` or your custom hostname.

Over SSH:

```bash
hostname
ip addr
ip route
```

#### Check services (systemd)

Main service unit name from the repo:

* `autostream.service` (see `system/systemd/autostream.service`)

Useful commands:

```bash
systemctl status autostream.service
journalctl -u autostream.service --no-pager -n 200
sudo systemctl restart autostream.service
```

Wi-Fi/AP mode service:

* `autostream_wifi_watcher.service` (see `system/systemd/autostream_wifi_watcher.service`)

```bash
systemctl status autostream_wifi_watcher.service
journalctl -u autostream_wifi_watcher.service --no-pager -n 200
sudo systemctl restart autostream_wifi_watcher.service
```

DNS/DHCP helper for AP mode (used during hotspot setup):

* `autostream_dnsmasq.service` (see `system/systemd/autostream_dnsmasq.service` and `system/dnsmasq/autostream-setup.conf`)

```bash
systemctl status autostream_dnsmasq.service
journalctl -u autostream_dnsmasq.service --no-pager -n 200
sudo systemctl restart autostream_dnsmasq.service
```

Other repo-provided units you may see enabled (depending on install):

* `autostream_rebooter.service` / `.path`
* `autostream_sdcardhealth.service` + `.timer`
* (see `system/systemd/`)

---

### Validate configuration

The main service launches:

* `autostream_webui.py` with:

  * `/opt/autostream/autostream.ini`
  * (from `ExecStart=` in `system/systemd/autostream.service`)

So the first thing to verify is that the config exists and is readable:

```bash
ls -l /opt/autostream/autostream.ini
```

If the UI is up but behavior is wrong, inspect `autostream.ini` and compare against what the code expects in:

* `core/autostream_config.py` (parsing + defaults)
* Any “setup”/settings handlers in:

  * `core/autostream_webui_pages.py`
  * `core/autostream_wifi_watcher.py` (hostname + Wi-Fi provisioning)

Hostname (device name) check:

```bash
cat /etc/hostname
hostname
```

Wi-Fi “configured SSID marker” check (used to decide whether to enter AP mode):

```bash
ls -l /opt/autostream/ssid
cat /opt/autostream/ssid
```

> If you suspect the Wi-Fi configuration state is “stuck”, the most reliable recovery path supported by the code is still: **reboot → join the `autostream_XXXX` hotspot → re-provision at `/setup`**.
