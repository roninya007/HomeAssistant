# Connecting an Akuvox Door Phone (R20 series) to Home Assistant

Last checked: 19 September 2026

This guide covers a local setup with no cloud account. It gives you a live camera view, a button that opens the door or gate, and a doorbell trigger for notifications.

## 1. What to expect

Home Assistant has no official Akuvox integration. You connect the device in separate parts:

| Goal | Method | Status |
|---|---|---|
| Live video and snapshot | Generic Camera (RTSP + JPEG snapshot) | Paths documented for R20A. Test on your model. |
| Open door or gate | Akuvox HTTP command via `rest_command` | Documented by Akuvox. |
| Doorbell press | Device HTTP push action calls a Home Assistant webhook | Reported by a forum user on an E12. Menu path may differ. |
| Two-way audio | SIP (needs a SIP server and app) | Optional. Not covered in detail. |
| All-in-one alternative | "Local Akuvox" custom integration (HACS) | R20K not on its supported list. Unverified. |

Verification notes:

- Confirmed on the device (Status > Basic > Product Information, checked 2026-09-19): Model **R20K**, MAC `0C:11:05:31:EF:48`, Firmware `320.30.11.37`, Hardware `320.1`, IP `192.168.7.25` (same `/24` subnet as the HA hub at `192.168.7.22` — an earlier reported IP of `192.168.1.100` was wrong/stale). DHCP reservation for `192.168.7.25` set on the router (TP-Link Archer) 2026-09-19.
- Firmware's third version number is 11 (≥10), so this device supports high security mode. Confirmed 2026-09-19: **High Security Mode is ON**, so use the section 3.5 URL form for high security mode (with or without credentials in the URL).
- Menu names change between firmware versions. Where this guide gives a menu path, treat it as a starting point and look for the closest match.
- I did not test any of this on your hardware.

## 2. Before you start

You need:

- The device on your network by Ethernet, with power (PoE or the supply your installer used).
- Admin login for the device web page. Ask your installer if you do not have it.
- A Home Assistant instance (Core, OS, or Container) on the same network or VLAN.
- A way to edit `configuration.yaml` (File editor add-on, Studio Code Server, or SSH).

Set a fixed address for the device. Either set a static IP on the device, or create a DHCP reservation on your router. Every step below uses this address, so it must not change. This guide writes it as `DEVICE_IP`.

## 3. Prepare the device

### 3.1 Open the web page

Enter `http://DEVICE_IP/` in a browser and log in as admin. If you do not know the IP, check your router's client list, or use the Akuvox IP scanner tool from the Akuvox website.

### 3.2 Understand the three separate logins

Akuvox keeps three independent credential sets:

1. The web page login.
2. The RTSP login (video stream and snapshot).
3. The HTTP API login (door opening).

Changing one does not change the others. If the camera works but the door will not open, the cause is often a mismatch here ([TapHome guide](https://taphome.com/en/docs/integration/sip-doorbell/akuvox/)). Write down all three as you set them. Using the same user name and password for RTSP and HTTP API makes later steps simpler.

### 3.3 Check the firmware and security mode

1. Go to Status > Basic > Product Information and read the firmware version.
2. If the third number in the version is 10 or higher, the device supports "high security mode".
3. Go to Security > Basic > High Security Mode and note whether it is on.

This decides which door-opening URL you use in section 3.5.

### 3.4 Enable video (RTSP)

1. Find the RTSP settings in the device menu. The location varies by firmware. Look for a page named RTSP, Monitor, or Video.
2. Enable the RTSP server.
3. Turn RTSP audio off. Audio is handled by SIP if you want it.
4. Set an RTSP user name and password.

For the R20A, TapHome documents these paths (port 554 is the standard RTSP port):

- Main stream, 1280x720: `rtsp://USER:PASS@DEVICE_IP:554/live/ch00_0`
- Sub stream, 640x480: `rtsp://USER:PASS@DEVICE_IP:554/live/ch00_1`
- Snapshot: `http://DEVICE_IP:8080/picture.jpg`

The same RTSP login protects the snapshot. I have not confirmed these paths on the R20K, so test them in section 4.

### 3.5 Enable door opening by HTTP

1. Go to Intercom > Relay.
2. Find "Open Relay via HTTP" and enable it.
3. Set a user name and password for HTTP access.
4. Leave "Session Check" disabled.
5. Set the relay delay (how many seconds the relay stays closed) to suit your lock or gate motor. Save.

Akuvox documents these URL forms ([Akuvox docs](https://knowledge.akuvox.com/docs/open-door-via-http-command.md)):

- High security mode off: `http://DEVICE_IP/fcgi/do?action=OpenDoor&UserName=USER&Password=PASS&DoorNum=1`
- High security mode on, credentials in the URL: `http://USER:PASS@DEVICE_IP/fcgi/OpenDoor?action=OpenDoor&DoorNum=1`
- High security mode on, no credentials in the URL: `http://DEVICE_IP/fcgi/OpenDoor?action=OpenDoor&DoorNum=1` (the device then relies on its own access rules)

Use `DoorNum=1` for Relay A and `DoorNum=2` for Relay B. Security relays use `DoorNum=SA` or `DoorNum=SB`.

If the door will not open but the camera works, turn high security mode off, or switch to the second URL form.

### 3.6 Check which relay your lock uses

Look at how the installer wired the lock or gate motor to the device. It sits on Relay A or Relay B (some models also have security relays). Use the matching `DoorNum`.

## 4. Test from a computer first

Test from a computer on the same network before touching Home Assistant. A failure here is easier to read than one inside Home Assistant.

Warning: the door test below opens the real door or gate. Make sure it is safe to do so.

Test the door command:

```bash
curl -v "http://DEVICE_IP/fcgi/do?action=OpenDoor&UserName=USER&Password=PASS&DoorNum=1"
```

Test the snapshot (the auth type is not confirmed, so `--anyauth` lets curl choose):

```bash
curl --anyauth -u USER:PASS -o test.jpg "http://DEVICE_IP:8080/picture.jpg"
```

Test the video stream in VLC (Media > Open Network Stream) or with ffplay:

```bash
ffplay -rtsp_transport tcp "rtsp://USER:PASS@DEVICE_IP:554/live/ch00_0"
```

If one of these fails, fix it here before moving on.

## 5. Add the camera to Home Assistant

1. Go to Settings > Devices & services > Add integration.
2. Search for "Generic Camera" and select it.
3. Fill in the form:
   - Still image URL: `http://DEVICE_IP:8080/picture.jpg`
   - Stream source URL: `rtsp://USER:PASS@DEVICE_IP:554/live/ch00_1`
   - RTSP transport protocol: TCP
   - Authentication: Basic (try Digest if the snapshot fails)
   - Username and Password: your RTSP login
   - Verify SSL certificate: off
4. Submit. Home Assistant shows a preview. Confirm it looks right.
5. Name the camera, for example `Front door`. Its entity ID will look like `camera.front_door`.

Start with the sub stream (`ch00_1`) for dashboards, as it uses less bandwidth. Switch to `ch00_0` if you need sharper video.

Alternative: the forum user with an Akuvox E12 added video through the ONVIF integration instead ([HA community thread](https://community.home-assistant.io/t/door-intercom-akuvox-r20k-on-ethernet/589574)). I could not confirm ONVIF support on the R20K. Try it only if the Generic Camera route fails.

## 6. Add the door-open button

### 6.1 Store the URL as a secret

Edit `secrets.yaml` and add one line (use your real values):

```yaml
akuvox_open_url: "http://DEVICE_IP/fcgi/do?action=OpenDoor&UserName=USER&Password=PASS&DoorNum=1"
```

### 6.2 Define the command and a script

Add this to `configuration.yaml`:

```yaml
rest_command:
  akuvox_open_door:
    url: !secret akuvox_open_url
    method: get
    timeout: 10

script:
  open_front_door:
    alias: Open front door
    icon: mdi:door-open
    sequence:
      - action: rest_command.akuvox_open_door
```

Older Home Assistant versions use `service:` in place of `action:`.

### 6.3 Reload

Go to Developer tools > YAML > Check configuration. If it passes, restart Home Assistant. Reloading only `rest_command` and scripts is possible, but a restart is the safest way to apply a new `rest_command` block the first time.

### 6.4 Test

Go to Developer tools > Actions, choose `script.open_front_door`, and run it. The door should open. If it does not, check Settings > System > Logs for the HTTP status code:

- 401 or 403: wrong credentials, or the wrong URL form for your security mode.
- Timeout: wrong IP, or a network or firewall block.
- 200 but no relay click: wrong `DoorNum`, or Open Relay via HTTP is disabled.

## 7. Add a dashboard card

Add a Picture Glance card or a vertical stack in the dashboard editor. Example YAML:

```yaml
type: vertical-stack
cards:
  - type: picture-entity
    entity: camera.front_door
    camera_view: live
    show_name: true
    show_state: false
  - type: button
    name: Open front door
    icon: mdi:door-open
    tap_action:
      action: perform-action
      perform_action: script.open_front_door
      confirmation:
        text: Open the door?
```

The confirmation prompt guards against accidental taps. Remove it if you prefer speed.

## 8. Doorbell press as a trigger

The idea: when someone presses the bell, the door phone sends an HTTP request to Home Assistant. A webhook automation catches it.

### 8.1 Create the automation

Pick a long random webhook ID, since the ID is the only secret. Do not use the sample below as is.

```yaml
automation:
  - alias: Doorbell pressed
    triggers:
      - trigger: webhook
        webhook_id: CHANGE-ME-long-random-string
        allowed_methods:
          - GET
          - POST
        local_only: true
    actions:
      - action: notify.mobile_app_YOUR_PHONE
        data:
          title: Doorbell
          message: Someone is at the door.
          data:
            image: /api/camera_proxy/camera.front_door
```

Replace `notify.mobile_app_YOUR_PHONE` with your phone's notify service (see Developer tools > Actions and search `notify.mobile_app`). Older versions use `platform:` in place of `trigger:` and `service:` in place of `action:`.

### 8.2 Point the device at the webhook

On the door phone, look for the push button action setting. On an E12, a forum user found it at Intercom > Basic > Push Button Action (bottom section), with the action set to HTTP ([HA community thread](https://community.home-assistant.io/t/door-intercom-akuvox-r20k-on-ethernet/589574)). The R20K may put it elsewhere. Search the device menu for "Action", "Push Button", or "HTTP URL".

Enter this URL as the HTTP target:

```
http://HOME_ASSISTANT_IP:8123/api/webhook/CHANGE-ME-long-random-string
```

Use the same string as the webhook ID. If you run Home Assistant behind HTTPS on a custom port or domain, use that address instead. Give Home Assistant a static IP too.

### 8.3 Test

Press the bell button. Check Settings > Automations > Doorbell pressed > Traces. A new trace confirms the webhook arrived.

If no trace appears, check that the device can reach Home Assistant on port 8123 (same VLAN, or a firewall rule that allows it), and that the URL has no typo.

## 9. Option B: the Local Akuvox custom integration

The [Local Akuvox integration](https://github.com/tykeal/homeassistant-local-akuvox) offers local control over the device's HTTP API. It creates a lock entity per relay and can manage users, PIN codes, and schedules.

Limits to know before you try it:

- It needs Home Assistant 2026.7.0 or later.
- Its README names E21V and R29 "or similar" models. The R20K is not listed, so I do not know if it works.
- Lock entities can unlock only. They cannot lock.
- Cloud-provisioned users and schedules cannot be changed locally.
- State updates every 30 seconds.
- Webhook events should use HTTPS, since HTTP sends PIN codes in plain text.

Install steps:

1. Install HACS if you do not have it.
2. In HACS, open the three-dot menu, choose Custom repositories, and add `https://github.com/tykeal/homeassistant-local-akuvox` with the category Integration.
3. Search for "Local Akuvox" in HACS, install it, and restart Home Assistant.
4. Go to Settings > Devices & services > Add integration and search for "Local Akuvox".
5. Follow the setup steps: enter the device IP or host name, choose whether to use SSL and certificate checks, choose the authentication type (None with IP allow list, Basic, or Digest), enter credentials if needed, and choose whether to enable webhook events.
6. On the device, enable the HTTP API and set the matching authentication mode. The README does not give device menu paths, so look in the security or API settings.

You can run this next to the camera setup from section 5. It replaces only the `rest_command` part.

Note on the [SmartPlus integration](https://github.com/nimroddolev/akuvox): it works through Akuvox's cloud and needs a SmartPlus account. Use it only if the installer manages your device in SmartPlus and you want that route.

## 10. Two-way audio (optional)

Home Assistant has no built-in SIP client, so talking to the visitor from a dashboard is not simple. Two common routes:

- Use the Akuvox app (SmartPlus) for calls, and Home Assistant for video, unlocking, and alerts. This is the easiest route.
- Run a SIP server (the forum user used the free tier of 3CX), register the door phone as an extension, and answer calls on a SIP phone or app. On the device, set the SIP account (user name, server address, transport UDP, PCMU codec, auto answer on if you want hands-free) as in the [TapHome guide](https://taphome.com/en/docs/integration/sip-doorbell/akuvox/). Status should show "Registered".

## 11. Security

- Keep the door phone on your LAN. Do not forward its ports to the internet.
- Change the default web password.
- Keep the door URL (with its password) in `secrets.yaml`, not in dashboards or shared files.
- Use `local_only: true` on the webhook, and a long random ID.
- Think about who can reach the "open door" script. Anyone with access to your Home Assistant dashboard can open the door. Limit the script to admin users, or add the confirmation prompt from section 7.
- If your network has a separate IoT or camera VLAN, allow only these paths: Home Assistant to the device (ports 80, 554, 8080), and the device to Home Assistant (port 8123).

## 12. Troubleshooting

| Symptom | Likely cause | What to try |
|---|---|---|
| Camera preview is blank | Wrong RTSP path or login | Test in VLC (section 4). Try `ch00_1`. Check the RTSP login. |
| Snapshot fails but stream works | Wrong auth type | Switch Basic and Digest in the camera setup. Confirm port 8080. |
| Stream drops or stutters | UDP transport or weak network | Set the RTSP transport to TCP. Use the sub stream. |
| Door does not open, camera works | Mismatched credentials, or high security mode | Match the HTTP API login. Try the other URL form. Turn off high security mode. |
| HTTP 401 or 403 | Wrong login | Re-enter the HTTP API credentials. |
| Door opens but for too short or too long | Relay delay | Change the delay in the relay settings. |
| Webhook never fires | Device cannot reach Home Assistant | Check network rules, the URL, and the ID. |
| Everything stopped after a router reboot | IP address changed | Use a static IP or DHCP reservation for both devices. |
| Changed the web password and video broke | Separate credential sets | Update the RTSP and HTTP API logins too. |

## 13. Final checklist

- [ ] Device has a fixed IP.
- [ ] Three logins recorded (web, RTSP, HTTP API).
- [ ] RTSP enabled. Stream and snapshot tested from a computer.
- [ ] Open Relay via HTTP enabled. Door command tested from a computer.
- [ ] Generic Camera added in Home Assistant.
- [ ] `rest_command` and script added. Script tested.
- [ ] Dashboard card added.
- [ ] Doorbell webhook automation created and tested.
- [ ] Door phone not exposed to the internet.

## Sources

- [Door Intercom - Akuvox R20K (on Ethernet), Home Assistant Community](https://community.home-assistant.io/t/door-intercom-akuvox-r20k-on-ethernet/589574)
- [Akuvox Doorbell Integration Guide, TapHome](https://taphome.com/en/docs/integration/sip-doorbell/akuvox/)
- [Open the Door via HTTP Command, Akuvox Knowledge Base](https://knowledge.akuvox.com/docs/open-door-via-http-command.md)
- [Local Akuvox Integration for Home Assistant (tykeal)](https://github.com/tykeal/homeassistant-local-akuvox)
- [Local Akuvox intercom control, Home Assistant Community](https://community.home-assistant.io/t/local-akuvox-intercom-control/993530)
- [Akuvox SmartPlus integration (nimroddolev)](https://github.com/nimroddolev/akuvox)
- [Akuvox R20 Series Door Phone Admin Guide](https://knowledge.akuvox.com/docs/r20-door-phone-admin-guide)
