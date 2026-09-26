# BYD + Nuki + Garmin — Claude Code implementation runbook

Prepared: 2026-09-17. Owner: Roni.

## 1. Mission and how to use this file

Configure or repair Home Assistant so a Garmin Fenix 7 running HassControl can lock/unlock a BYD ATTO 3 and a Nuki Smart Lock Ultra. Use the existing working installation wherever possible. This is an implementation handoff, not evidence that you have access to the live system.

Suggested instruction to Claude Code:

> Read this runbook. Inspect my actual Home Assistant installation and identify what already works. Back up the affected configuration, implement the missing BYD/Nuki/Garmin configuration, validate it, and report the results. Continue autonomously with reversible configuration work using the access I provide. Ask me only for missing access, credentials, phone/watch interactions, or an attended physical lock test. Preserve working integrations and unrelated configuration. Do not issue physical lock/unlock commands until we agree to perform that test.

This document is based on `Smart_Home_Project_Knowledge(1)(1).md`, dated 2026-09-12, and the project conversation. Additional upstream references are identified below. It contains no real passwords, tokens, or vehicle PINs.

### Evidence and uncertainty

- **Previously working:** Home Assistant OS in VirtualBox; HACS; BYD custom integration; BYD watch control; Mosquitto; Nuki MQTT; `lock.front_door`; separate Nuki scripts. Roni subsequently confirmed the Nuki setup worked.
- **Known historical entities:** `lock.byd_atto_3_lock`, `lock.front_door`, `script.lock_home`, `script.unlock_home`.
- **Not recorded:** exact installed BYD repository/version, current IP addresses, Garmin HTTPS URL, token, final BYD script contents, MQTT credentials, and remote-access method.
- **Proposed standardization, not a recovered final configuration:** the four-script example below. Preserve existing working BYD controls unless changes are needed.
- **New hosting is unconfirmed:** a later conversation proposed an old Samsung S20 FE 5G as a server. There is no evidence here that migration succeeded. Discover the actual host; do not assume VirtualBox or Android.

## 2. Architecture and intended controls

Garmin Fenix 7 → HassControl → paired phone / Garmin Connect → Home Assistant.

From Home Assistant:

- BYD custom integration → BYD cloud → ATTO 3.
- MQTT integration → Mosquitto broker → Nuki Ultra over home Wi-Fi.

The Nuki MQTT design does not need a Nuki Bridge or Matter hub. Home Assistant and its broker must remain running. The old laptop setup stops when the laptop or VM shuts down.

| Watch label | Suggested script | Physical target | Action |
|---|---|---|---|
| Lock BYD | `script.lock_byd` | `lock.byd_atto_3_lock` | `lock.lock` |
| Unlock BYD | `script.unlock_byd` | `lock.byd_atto_3_lock` | `lock.unlock` |
| Lock Home | `script.lock_home` | `lock.front_door` | `lock.lock` |
| Unlock Home | `script.unlock_home` | `lock.front_door` | `lock.unlock` |

Only ordinary lock/unlock is in scope. Do not add Unlatch, `lock.open`, Lock 'n' Go, automatic unlocking, or unrelated vehicle commands.

## 3. Start with discovery and access

Before editing, collect a sanitized inventory:

1. Home Assistant installation type, version, host, and actual configuration directory. `/config` is a common HA-side path; it is not automatically available on the machine running Claude Code.
2. Available access: configuration filesystem/SSH, Home Assistant UI, API credentials, and Windows/VirtualBox management if applicable. A Home Assistant API token alone does not provide arbitrary filesystem access.
3. Existing script/group includes in `configuration.yaml`, scripts, Garmin group, HACS entry, BYD integration version and entity IDs, MQTT integration and broker status.
4. Actual phone/watch HassControl settings, obtained without printing the token.
5. Network addresses: Home Assistant URL reachable from Claude, broker address reachable from Nuki, and HTTPS Home Assistant URL reachable from the paired phone. These can be different.
6. Whether live physical testing is currently authorized and someone can verify both locks.

Read integration metadata and configuration locally without dumping credentials or all of `.storage`. Use supported UI/configuration flows for integration setup; do not manufacture config entries by editing `.storage`.

If Claude runs in a separate Ubuntu development VM, its `localhost` is that VM, not the Windows host or Home Assistant VM. Identify a reachable address first.

Back up affected YAML files and broker settings, and use the existing Home Assistant backup mechanism where available. Record the backup location, versions, and original network settings. Keep secret-bearing backups private and outside public Git repositories.

## 4. Home Assistant and network prerequisites

### Historical VirtualBox installation

The project originally used HA OS on Windows/VirtualBox with NAT. The notes mention host port 8123 forwarding to guest 8123, while another conversation confirmed `http://127.0.0.1:8080` worked on Windows. This suggests the host mapping changed. Inspect the actual forwarding rule; do not hardcode either host port.

`127.0.0.1` only works on the machine owning that listener. It cannot be copied into the Nuki or phone settings.

For MQTT, the successful NAT mapping was:

| Rule | Protocol | Host IP | Host port | Guest IP | Guest port |
|---|---|---|---|---|---|
| MQTT | TCP | blank | 1883 | blank | 1883 |

A loopback-only host bind prevents the physical Nuki from reaching the broker. Windows Firewall should allow TCP 1883 on the trusted Private network, scoped to the local subnet. Inspect and reuse an existing rule rather than duplicating it. Do not change the network profile blindly or forward MQTT from the internet through the router.

Read-only Windows diagnostics:

```powershell
Get-NetIPConfiguration
ipconfig
Test-NetConnection -ComputerName <BROKER_LAN_ADDRESS> -Port 1883
```

Use the physical Wi-Fi/Ethernet adapter's home-LAN address. The previously observed `192.168.56.1` was a VirtualBox host-only address, not the Nuki broker destination. A successful TCP probe proves port reachability only; it does not prove MQTT authentication. A probe from another LAN device gives stronger evidence of the Nuki's network path than a host-local probe.

### Other hosts

On a bridged VM or dedicated host, Nuki normally targets the broker's directly reachable LAN address; the Windows NAT rules are unnecessary. Discover the topology before changing it. If using a container or another installation without HA apps/add-ons, Mosquitto requires separate provisioning; do not assume the HA OS add-on workflow exists. Keep a stable broker address, preferably using a router DHCP reservation when router access is available.

## 5. BYD integration

### Existing installation: preferred route

1. Find the BYD integration in HACS and Settings → Devices & services. Record its repository/version.
2. Confirm the BYD account can see and control the ATTO 3 in the BYD phone app.
3. Resolve the actual lock entity. The historical ID is `lock.byd_atto_3_lock`; do not create a fake entity if the current ID differs.
4. Inspect the installed integration's command/PIN requirements and current scripts. Determine whether the PIN is stored in the integration or passed as `data.code`.
5. Preserve a functioning account, region, integration and Garmin connection.

### Fresh setup when no integration exists

The historical notes do not identify the repository. One currently documented candidate is [jkaberg/hass-byd-vehicle](https://github.com/jkaberg/hass-byd-vehicle); verify suitability before selecting it. Its documented setup is:

1. In HACS, add its repository as an Integration under Custom repositories, install BYD Vehicle, and restart HA.
2. Add BYD Vehicle through Settings → Devices & services.
3. Complete the UI setup with BYD credentials, correct country and control PIN. Set the operation PIN in the BYD app first.
4. A dedicated BYD account is recommended by that project to avoid disrupting the main app session; it must already have access to the vehicle and its own control PIN.

If HACS is absent, use its current official installation instructions rather than an unverified shell installer. Account creation/sharing may need Roni's phone interaction. Do not claim a specific sharing path was established in the old sessions.

### The earlier BYD bug

An earlier `lock_byd` script contained BOTH a lock action and an unlock action in the same sequence. Running it executes both, ending with unlock. An action alias such as “Unlock BYD” labels a sequence step; it does not create a separate watch command.

Inspect the current script before changing it. If the faulty sequence remains, split it into two explicit actions and update references. Do not invoke the faulty sequence to diagnose it. The exact final BYD fix was not preserved in the notes.

## 6. Nuki Ultra through MQTT

### A. Broker and Home Assistant

For the historical HA OS installation:

1. Install the official Mosquitto broker app/add-on if missing.
2. Create/reuse a dedicated Nuki broker login. Merge this pattern into its existing configuration; do not replace other options:

```yaml
logins:
  - username: nuki_mqtt
    password: '<SET_A_UNIQUE_PASSWORD_PRIVATELY>'
```

3. Keep only one `logins` key; remove a conflicting empty `logins: []` only when merging. Preserve existing users.
4. Save and restart Mosquitto after changes. Enable its startup/watchdog options if available and appropriate.
5. Configure/accept the MQTT integration in Settings → Devices & services. Ensure Home Assistant itself connects to the broker. Do not create a second competing MQTT setup.

The example is broker add-on configuration, not an entry for `configuration.yaml`. Do not assume Home Assistant `!secret` tags work in the add-on's settings editor. Use the supported credential mechanism there. Check the current Nuki firmware's credential limits if it rejects input; no exact limits were recorded.

### B. Nuki app — user interaction may be required

1. Confirm the Ultra is connected to home Wi-Fi.
2. Open its MQTT settings in the Nuki app. Menu labels may vary with firmware/app version.
3. Enter the reachable broker LAN hostname/IP, port `1883`, and dedicated broker credentials.
4. For the recorded NAT setup, enter the Windows host's LAN address, not the internal VM address.
5. Do not put `http://`, `mqtt://`, `localhost`, or `:8123` in the broker hostname field.
6. Enable MQTT, permit MQTT control if a separate setting exists, and enable Home Assistant discovery if offered.
7. Verify the app reports successful activation and inspect the Mosquitto log for the Nuki connection.

MQTT on port 1883 is the recorded local-network setup. Keep it on the trusted LAN. Do not invent MQTT topics or retained commands to force discovery.

### C. Discovery

The working installation discovered a Front Door device under MQTT. Its primary entity was `lock.front_door`. Other entities included Unlatch and Lock 'n' Go; leave these out of the watch group.

If discovery fails, inspect broker connection, HA MQTT discovery settings, Nuki discovery settings and logs before creating manual entities. Once discovered, confirm the main entity is available. Perform physical lock/unlock tests only in the attended test phase.

## 7. YAML configuration

These are merge templates. Resolve real entity IDs first. Do not overwrite unrelated scripts, duplicate top-level keys, or change an established include structure.

### A. Script include

If the installation uses the usual separate scripts file, `configuration.yaml` includes:

```yaml
script: !include scripts.yaml
```

If scripts are inline, packaged, or split across directories, adapt the changes to that structure. In `scripts.yaml`, script keys do not have the `script.` prefix and there is no outer `script:` wrapper.

### B. Four independent scripts

The Nuki pair follows the previously working pattern. The BYD pair is a corrected explicit-action template using the historically observed `code` argument. If the installed integration uses its stored PIN and does not need/accept `code`, omit the `data` block after verifying its requirements.

```yaml
lock_byd:
  alias: Lock BYD
  mode: single
  sequence:
    - action: lock.lock
      target:
        entity_id: lock.byd_atto_3_lock
      data:
        code: !secret byd_control_pin

unlock_byd:
  alias: Unlock BYD
  mode: single
  sequence:
    - action: lock.unlock
      target:
        entity_id: lock.byd_atto_3_lock
      data:
        code: !secret byd_control_pin

lock_home:
  alias: Lock Home
  mode: single
  sequence:
    - action: lock.lock
      target:
        entity_id: lock.front_door

unlock_home:
  alias: Unlock Home
  mode: single
  sequence:
    - action: lock.unlock
      target:
        entity_id: lock.front_door
```

If using `!secret`, add the actual PIN privately to the applicable HA `secrets.yaml`, preserving any existing entries:

```yaml
byd_control_pin: '<REAL_PIN_AS_A_QUOTED_STRING>'
```

Do not deploy that placeholder as a real PIN. Keep the value quoted to preserve leading zeroes. `secrets.yaml` separates credentials from ordinary YAML; it is not encryption. Never include its contents in chat, diffs, or Git. See [HA secrets documentation](https://www.home-assistant.io/docs/configuration/secrets/) and [script syntax](https://www.home-assistant.io/docs/scripts/).

`mode: single` applies to each script individually; it is not a shared interlock between the lock and unlock scripts. Do not send conflicting commands simultaneously.

### C. Garmin group

The historically working group exposed the BYD lock directly alongside the two Nuki scripts:

```yaml
group:
  garmin:
    name: Garmin
    entities:
      - lock.byd_atto_3_lock
      - script.lock_home
      - script.unlock_home
```

Keep that layout if BYD already works and only Nuki needs adding. For an intentionally standardized four-button setup, use the following instead, after validating all four scripts:

```yaml
group:
  garmin:
    name: Garmin
    entities:
      - script.lock_byd
      - script.unlock_byd
      - script.lock_home
      - script.unlock_home
```

Merge with other intended watch entries. Avoid adding both a direct lock and its scripts accidentally. If `configuration.yaml` already contains `group: !include groups.yaml`, put the `garmin:` section in `groups.yaml` without the outer `group:` key.

Every entity needs its own list item. A prior broken edit concatenated three entity IDs into one string and produced an “invalid entity ID” notification.

## 8. Validate and load Home Assistant configuration

1. Review the diff, checking duplicate keys, entity IDs, includes and unresolved placeholders. Redact secrets.
2. Run Home Assistant's own configuration check. On HA OS with its CLI available, use `ha core check`; otherwise use the installation's supported check or Developer tools → YAML check. A generic YAML parser alone cannot validate HA includes, tags or semantics.
3. Fix errors before restarting. Reload scripts/groups through the supported UI/actions when available; otherwise restart HA after a successful check. Do not reboot the entire host unnecessarily.
4. Confirm the relevant script entities and `group.garmin` loaded, and inspect logs for errors.
5. If Developer tools is hidden, open the user profile and enable Advanced Mode, then return to the sidebar. UI placement can vary.

Scripts appearing as “Ungrouped” under Settings → Devices & services → Entities is normal; they are logical controls, not automatically members of the physical Nuki device.

## 9. Garmin / HassControl setup

Preserve existing working settings. For a fresh setup, follow the [HassControl maintainer instructions](https://github.com/hasscontrol/hasscontrol):

1. Pair the Fenix 7 with Garmin Connect and install HassControl via Connect IQ.
2. Open HassControl's settings in the phone's Connect IQ app.
3. Set Host to a phone-reachable Home Assistant **HTTPS** URL. HassControl documents HTTPS as required. The laptop's local HTTP/loopback URL is not this endpoint.
4. Authenticate through its supported sign-in flow or enter a Home Assistant long-lived access token privately.
5. Set Group to `group.garmin`.
6. Open HassControl on the watch, access its menu (press-and-hold/menu gesture as appropriate), then Settings → Refresh entities.
7. If controls are missing, check the start-view filter; its default scenes view can hide other entity types. Keep Garmin Connect running and the phone paired/in range.

The project notes do not record how HTTPS or away-from-home access was configured. Discover the working method. If absent, report this as a prerequisite and implement an agreed supported access solution; do not expose raw HTTP or MQTT ports publicly as a shortcut.

The phone's connection must reach HA wherever watch control is expected to work. Home Wi-Fi success alone does not prove mobile-data access.

## 10. Attended end-to-end tests

Preparation and read-only checks may proceed without actuating anything. For physical tests, have Roni present with a recovery method for the home lock and access to the parked vehicle. Agree on the intended final lock states. Do not repeatedly cycle locks as a connectivity test.

Test one command at a time:

| Layer | Check | Evidence |
|---|---|---|
| Configuration | HA check passes and scripts/group exist | Check result; entity/group state |
| Nuki integration | Main lock available | MQTT device/entity and broker connection |
| BYD integration | Vehicle lock available | Integration/entity state; no auth errors |
| HA scripts | Each approved command performs only its labeled action | Script trace plus physical confirmation |
| Watch | Each intended control appears and runs the correct script/action | Watch observation, HA trace, physical result |
| Remote path, if required | Repeat a selected approved test while phone uses mobile data | Phone reaches HTTPS endpoint; physical result |

An API success response or a completed script trace is not proof of physical locking. Inspect device-reported state and obtain physical confirmation. Allow for BYD cloud update latency; avoid aggressive polling/retries. Finish in the states agreed with Roni.

## 11. Troubleshooting from the actual sessions

| Symptom | Evidence / likely issue | Next step |
|---|---|---|
| BYD “Lock” ends unlocked | Earlier script had lock followed by unlock | Inspect sequence and watch target; separate scripts |
| Nuki activation error 8E | In our session, no broker connection appeared | Check address, routing, Wi-Fi isolation, NAT and firewall |
| Nuki error 89 with credential message | In our session, broker was reached but auth failed | Check exact credentials, single `logins` block, restart broker |
| Probe connects then disconnects | TCP test without MQTT login | Expected for `Test-NetConnection`; not proof of auth |
| Repeated short internal connections | May be HA/add-on health checks | Correlate client/user and timing; do not assume Nuki |
| Invalid entity ID after YAML edit | Several IDs were on one line | Separate `- entity_id` entries; validate again |
| Scripts absent | Missing include, wrong file or failed reload | Check actual config path and logs |
| HA works, watch does not | URL/auth/group sync/filter/phone path | Verify each layer without changing device integration |
| Works until laptop sleeps | Server/broker stopped | Restore host availability; plan always-on hosting separately |
| MQTT stops after network change | Broker's host IP may have changed | Verify LAN IP, forwarding and Nuki destination |

The Nuki code interpretations above describe observed incidents, not a universal error-code specification. No broker log entry suggests a connectivity issue but is not conclusive if logging is disabled or incomplete.

## 12. Rollback and completion report

On a failed configuration change:

1. Restore only the modified files/settings from the recorded backup.
2. Validate restored configuration and reload/restart the affected component.
3. If group membership changed, refresh HassControl again.
4. Confirm prior entities/controls are restored. Restore network rules only if this task changed them.
5. Remember that configuration rollback does not restore physical lock states. Verify those separately with Roni.

At completion, report:

- Actual host, installation type, BYD repository/version and resolved entity IDs.
- Files/settings changed and backup location, with sanitized diff if useful.
- Watch group and available controls.
- Validation performed, distinguishing configuration checks from real physical tests.
- Any blocked step and the exact minimum user action needed.
- Agreed final physical lock states, if tested.

Do not declare end-to-end success from YAML validation alone. This file enables autonomous configuration work once access is supplied; it cannot supply credentials, pair a watch, operate phone-only setup screens, or prove physical results by itself.
