# Running Home Assistant on a Samsung Galaxy S20 FE 5G (as an always‑on hub)

This guide turns an old, unrooted Samsung S20 FE 5G into a 24/7 Home Assistant
host, using **Termux + udocker** to run the official Home Assistant Docker
image without root. It also documents what is *not* realistically possible on
this hardware, so you don't waste time chasing it.

## 0. Reality check — read this first

- **Home Assistant OS (HAOS) and Home Assistant Supervised cannot be installed
  on this phone.** HAOS ships as disk images for a specific list of supported
  boards/x86 systems with a standard bootloader; a Samsung phone's Android
  bootloader isn't on that list and there is no supported way to flash it.
- **A real Docker daemon needs root.** Docker requires creating kernel
  namespaces, writing cgroups, and mounting overlay filesystems — permissions
  a stock, unrooted Android app is never granted. So plain `docker` commands
  won't work in Termux.
- The practical workaround is **udocker**, a tool that *pulls* real Docker
  images and runs them without a daemon and without root (via `proot`). It
  behaves like a container for running Home Assistant, but you won't get
  `docker build`, `docker compose` networking, or daemon-level features.
- **USB Zigbee/Z-Wave dongles are unreliable on this setup.** Termux's
  `termux-usb` API exists, but tools that expect Linux-style USB/serial access
  (like Zigbee coordinator drivers) generally need root to work properly.
  Plan on **network-based integrations only** (ESPHome, Tasmota, MQTT, cloud
  APIs, local push integrations) rather than a USB radio stick.
- **No Supervisor add-on store.** You get Home Assistant itself; things like
  Mosquitto or Node-RED "add-ons" aren't available the normal way. You can
  still run separate containers/services for them, or use
  [HACS](https://hacs.xyz) for custom integrations/frontend cards, since HACS
  works independently of the Supervisor.
- Performance-wise the S20 FE 5G (Snapdragon 865 / Exynos 990, 6–8 GB RAM) is
  more than enough for Home Assistant Core managing a normal home. The real
  risks are **Android killing the background process** and **battery/flash
  wear from running 24/7** — both addressed below.

Source for the udocker method verified below:
[huytungst/HomeAssistant-Termux](https://github.com/huytungst/HomeAssistant-Termux).

## 1. Install Termux (not from Google Play)

The Play Store build of Termux is deprecated/frozen and missing functionality
compared to the maintained builds. Install from one of the official sources
instead:

- F-Droid: https://f-droid.org/packages/com.termux/
- or the GitHub releases: https://github.com/termux/termux-app/releases

Also install the companion app **Termux:Boot** from the same source
(F-Droid) — you'll need it in step 5 to auto-start Home Assistant after a
reboot.

## 2. Prepare the Termux environment

Open Termux and run:

```bash
pkg update && pkg upgrade -y
pkg install git -y
```

## 3. Get Home Assistant running via udocker

```bash
git clone https://github.com/huytungst/HomeAssistant-Termux.git
cd HomeAssistant-Termux
./install_udocker.sh
```

If the install script fails with an error like
`no image found in manifest for platform android/arm64 home assistant`
(a known issue as of mid‑2026), install udocker directly instead and re-run:

```bash
pkg install udocker -y
```

Then start Home Assistant:

```bash
./home-assistant-core.sh
```

First boot takes roughly 5–10 minutes while Home Assistant finishes setting
itself up. Once it's ready, open, from any device on the same network:

```
http://<phone-ip>:8123
```

(Find `<phone-ip>` under Android **Settings → About phone → Status →
IP address**, or `ifconfig` inside Termux.)

Optional: the same repo includes a Matter server script
(`./matter-server.sh`, run in a second Termux session) if you need Matter
device support — point Home Assistant's Matter integration at
`ws://localhost:5580/ws`.

## 4. Keep it running: Android will try to kill it

This is the step people skip and then wonder why their "hub" goes offline
overnight. Android's battery management (and Samsung's One UI layer
specifically) aggressively suspends background apps by default.

On the phone, go to **Settings → Apps → Termux → Battery** and set it to
**Unrestricted** (not "Optimized"). Do the same for **Termux:Boot** once
installed.

Also disable the two most common silent killers on Samsung devices:

- **Settings → Device care → Battery → Background usage limits** — make sure
  Termux is **not** in "Sleeping apps" or "Deep sleeping apps", and turn off
  **Put unused apps to sleep** / **Auto revoke permissions** for it if shown.
- **Settings → General management → Reset → Auto restart** (naming varies by
  One UI version; sometimes under **Device care → Auto optimization**) —
  turn this **off**. Samsung phones can be configured to silently reboot on a
  schedule "to keep performance optimal," which would kill your Termux
  session and, without Termux:Boot correctly configured, take Home Assistant
  down with it.

For background/process-limit specifics that change between Android and One UI
versions, [dontkillmyapp.com/samsung](https://dontkillmyapp.com/samsung) is a
maintained reference — check it if Termux keeps getting killed after doing
the above.

Additional hardening:

- Install **Termux:API** and run `termux-wake-lock` once inside your Termux
  session (prevents the CPU from deep-sleeping while Termux is running).
- In recent apps, long-press the Termux card and choose **Lock this app**
  (or the equivalent "keep open" toggle) so the OS won't swipe-kill it.
- Set up **Termux:Boot**: create `~/.termux/boot/start-homeassistant.sh`
  inside Termux containing `cd ~/HomeAssistant-Termux && ./home-assistant-core.sh`,
  and make it executable (`chmod +x`). This re-launches Home Assistant
  automatically after any reboot (scheduled or power-loss).

## 5. Keep it powered and connected

- Leave the phone on a charger permanently. If your One UI version has
  **Settings → Battery → Battery protection / Protect battery** (a charge
  cap, typically ~80%), turn it on — it noticeably reduces battery
  degradation from being always plugged in, at the cost of not running purely
  on internal battery for long during a power cut.
- Turn the screen brightness down and set a short screen timeout — the
  screen doesn't need to be on for Termux to keep running once wake-lock and
  battery-unrestricted are set.
- Disable Wi‑Fi power‑saving and any "turn off Wi‑Fi if no data" toggle
  (**Settings → Connections → Wi‑Fi → Advanced**), and disable automatic
  switching to mobile data.
- On your router, set a **DHCP reservation (static IP)** for the phone's MAC
  address, so `http://<phone-ip>:8123` never changes.
- Don't port-forward 8123 directly to the internet. For remote access, use
  Home Assistant's built-in **Remote UI (Nabu Casa)** or a private VPN mesh
  like **Tailscale**, and put a reverse proxy with TLS in front if you expose
  anything publicly.

## 6. First-run setup inside Home Assistant

Once `http://<phone-ip>:8123` loads:

1. Create your admin account and set your home's location/timezone.
2. Skip auto-discovered integrations you don't recognize; add the ones you
   actually use (network-based ones will work best on this setup — see the
   USB caveat in section 0).
3. Go to **Settings → System → General** and confirm the reported time zone
   and unit system.
4. Consider lowering **Settings → System → Storage / Recorder** history
   retention (`purge_keep_days`) — Home Assistant's default SQLite recorder
   writes frequently, and phone internal storage doesn't have the
   wear-leveling of an SD card or SSD. Fewer days of history, or excluding
   noisy entities from recording, reduces flash wear over a multi-year 24/7
   run.

## Known limitations to accept with this setup

- No Supervisor, so no one-click add-ons (Mosquitto, Node-RED, ESPHome
  dashboard, etc.) — you'd add these as separate services/containers if
  needed.
- No reliable USB Zigbee/Z-Wave coordinator support without root.
- Less battle-tested than a Raspberry Pi or mini PC running official HAOS —
  expect the occasional Termux/udocker quirk after Android or app updates.
- Constant charging and constant writes will wear the phone's battery and
  storage faster than normal phone use. This is a trade-off of repurposing
  old phone hardware, not a Home Assistant limitation.

## Sources

- [huytungst/HomeAssistant-Termux](https://github.com/huytungst/HomeAssistant-Termux) — udocker install steps, verified directly.
- [Termux GitHub discussion on official install sources (Play Store deprecated)](https://github.com/termux/termux-app/discussions/4000)
- [dontkillmyapp.com/samsung](https://dontkillmyapp.com/samsung) — Samsung-specific background/battery kill behavior.
- [udocker user manual](https://indigo-dc.github.io/udocker/user_manual.html) — capabilities/limits of rootless container execution.
- General Home Assistant architecture facts (HAOS supported-hardware model, no root ⇒ no real Docker daemon) are standard, publicly documented Home Assistant/Android platform behavior, not from a single cited article.
