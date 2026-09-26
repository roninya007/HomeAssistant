# BYD Car ↔ Home Assistant Integration — Setup Notes

Reference notes from setting up remote lock/unlock of a BYD car (Atto 3 / Dolphin / Seal,
MyBYD app) through Home Assistant. Kept for reuse on similar integrations later.

## Environment this was done on

- Home Assistant **Core** (not HAOS/Supervised) running on an old Samsung S20 FE 5G,
  via Termux (F-Droid build) → `proot-distro` Debian container, no root.
- HA config dir: `/root/homeassistant` (inside the Debian proot).
- HA venv: `/root/ha-venv-314` (Python 3.14, uv-managed). HA 2026.3+ requires
  Python >= 3.14.2; the original `/root/ha-venv` (Debian's Python 3.13, stuck on HA
  2026.2.3) is kept only for rollback.
- Phone LAN IP: `192.168.7.22`, HA on port `8123`.
- SSH into the phone: `ssh -p 8022 u0_a311@192.168.7.22` (Termux sshd, manually started
  by the user when needed — never added to boot scripts).
- On boot, `~/.termux/boot/start-homeassistant.sh` takes a wake lock and `exec`s
  `~/hass-watchdog.sh`, which starts and supervises both Mosquitto and HA (venv path is
  the `HA_VENV` variable at the top of the script).
  The `exec` chaining matters: proot has a `--kill-on-exit` behavior, so if you background
  a process (`&`/`nohup`) *inside* a `proot-distro login` shell and that shell exits, the
  background job dies too. Launch detached from the Termux side instead
  (`nohup proot-distro login debian -- ... &`), and `exec` all the way through.

## Manually restarting Home Assistant over SSH

Needed after installing or updating a custom integration. Easiest: Settings → System →
⋮ → Restart Home Assistant in the UI. Over SSH, just stop HA and let the watchdog
start it again within about a minute:

```bash
# Stop it by PID. Don't put the process pattern itself in the ssh command text:
# `pkill -f` would also match (and kill) the ssh session running it.
ssh -p 8022 u0_a311@192.168.7.22 'kill $(ps -eo pid,args | awk "/venv-314.bin.hass/ && !/awk/ {print \$1}")'

# Poll until it's back
curl -s -o /dev/null -w "%{http_code}" http://192.168.7.22:8123/
```

Don't let two HA instances run on the same config at once. When starting HA by hand,
stop the watchdog first.

## Installing HACS on HA Core (no Supervisor)

HA Core doesn't ship HACS or an add-on store, so it has to be installed manually via
the official installer script, run *inside* the Debian proot:

```bash
proot-distro login debian
```

The official one-liner (`https://get.hacs.xyz`) internally requires `wget`, which is
**not preinstalled** in a fresh Debian proot container — install it first:

```bash
apt-get update -qq && apt-get install -y -qq wget unzip
cd /root/homeassistant
wget -O - https://get.hacs.xyz | bash -
```

Then restart HA (see above), and finish setup in the UI:
**Settings → Devices & Services → Add Integration → HACS**, which walks through a
GitHub device-authorization flow (visit github.com/login/device, enter the shown code).

## HACS UI quirks (v2.0.x — the redesigned frontend)

Older HACS tutorials describe menus that have moved:

- **Custom repositories** is under the **⋮ (three-dot) menu in the top-right corner of
  the main HACS dashboard page** — not a per-tab menu, not under "Integrations" specifically.
- HACS's default store now actually **includes `jkaberg/hass-byd-vehicle`** — no need to
  add it as a custom repository at all. Just search "BYD" directly in HACS. (If you try
  to add it as custom anyway, HACS says "Repository exists in the store" — that's fine,
  just cancel and search instead.)

## Home Assistant UI quirks (this version)

- **Developer Tools** is not in the top-level sidebar by default. It only appears after
  enabling **Advanced Mode** (click your profile/avatar, bottom-left → toggle Advanced
  Mode). Even then, in this version it shows up **nested under Settings**, not as its
  own sidebar icon. Direct URL also works: `http://<ha-host>:8123/developer-tools/action`.
- Testing an action (e.g., a lock) without a dashboard card: **Developer Tools → Actions**,
  search the action (`lock.lock` / `lock.unlock`), pick the entity as target, run it.

## The BYD integration itself

- **Repository:** [jkaberg/hass-byd-vehicle](https://github.com/jkaberg/hass-byd-vehicle)
  — HACS custom integration, talks to BYD's cloud API via the `pyBYD` library. Fits
  EU/international BYD models paired with the **MyBYD** app (Atto 3, Dolphin, Seal).
  (Different from mainland-China BYD/DiLink app ecosystem — that'd need a different
  integration.)
- **Before configuring:** open the MyBYD app and set an **operation/control PIN**
  (Settings in the app). Remote lock/unlock commands fail without this even if login
  succeeds.
- **Recommended:** create a second BYD account (invited to the car in the app) for HA
  to use, so HA logging in doesn't kick your personal phone's MyBYD session out.
  Optional — using your main account works too, just expect occasional app logouts.
- **Config flow fields:** username/phone, password, **Country** (defaults to UK — must
  match your actual account's market or login fails), optional control PIN, climate
  duration, debug mode.
- **Entities created:** a `lock.*` entity for the doors, plus climate/status sensors.
- **Verified working:** `lock.lock` / `lock.unlock` via Developer Tools → Actions,
  confirmed against the physical car.
- **Known tradeoff:** frequent cloud polling can drain the car's 12V battery over time —
  consider lengthening the poll interval in the integration's options once it's stable.

## Remote access (accessing HA / the BYD lock from outside the home network)

Goal: reach HA (and the BYD lock control) from anywhere, via a custom subdomain
(`rha.nemesis.co.il`), under two hard constraints:
1. Don't touch `nemesis.co.il`'s main nameservers/existing DNS records — only adding a
   new subdomain record is acceptable.
2. Don't restructure the HA install itself, and the phone may roam between networks
   (home, work, etc.) rather than staying fixed on one router.

**Ruled out:**
- **Cloudflare Tunnel** — its automatic DNS routing (`cloudflared tunnel route dns`)
  needs Cloudflare to be the *authoritative DNS for the whole zone* (nameservers pointed
  at Cloudflare), which conflicts with constraint 1. (Cloudflare's free tier doesn't
  offer a subdomain-only/CNAME-only zone activation for this — that's an Enterprise/
  "Cloudflare for SaaS" feature.)
- **Dynamic DNS + router port-forward** (DuckDNS-style) — conflicts with constraint 2,
  since it assumes the phone stays on one network whose router you control. Doesn't
  work if the phone roams to a network (e.g. work Wi-Fi) where you can't configure
  port forwarding.

**Viable options identified (decision pending):**
- **Cheap VPS relay (leaning choice):** rent a small VPS (~$4-6/mo) with a static IP,
  add one A record (`rha.nemesis.co.il` → VPS IP), phone keeps an outbound tunnel
  (e.g. WireGuard) open to the VPS from wherever it is, VPS runs the HTTPS reverse
  proxy (e.g. Caddy) and forwards through the tunnel to HA. Outbound-only from the
  phone's side, so it works regardless of which network's router you don't control.
- **ngrok with a custom domain** (paid plan, ~$8-10+/mo): CNAME the subdomain to an
  ngrok-provided address at your existing DNS provider (no nameserver migration), run
  ngrok's agent on the phone pointed at `localhost:8123`. Less to manage than a VPS,
  more recurring cost, adds a vendor dependency.
- **Nabu Casa** (official HA Cloud, ~$6.5/mo): simplest of all, but gives a
  `xxxx.ui.nabu.casa` URL instead of the custom `rha.nemesis.co.il` subdomain — custom
  domains don't fit cleanly since Nabu Casa's TLS cert only covers its own hostname.
  Only fits if the custom-subdomain requirement is dropped.

## Companion app

- Official app is just called **"Home Assistant"** on the Play Store / App Store (free).
- On the same home Wi-Fi, point it at `http://192.168.7.22:8123` directly.
- Remote access (from outside the home network) needs one of the options above first.
