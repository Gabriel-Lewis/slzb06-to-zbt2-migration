# Migrating from SLZB-06 to Home Assistant Connect ZBT-2 on Zigbee2MQTT
## Coordinator Migration Without Re-Pairing Devices

This guide covers migrating a Zigbee network from the [SLZB-06](https://amzn.to/4ejFiYz) (ZNP/Z-Stack, network-attached) to the [Home Assistant Connect ZBT-2](https://ameridroid.com/products/home-assistant-connect-zbt-2) (EZSP/EmberZNet, USB) running Zigbee2MQTT on Home Assistant OS. No *removing* devices is required, but expect to manually wake most routers. Z-Stack → Ember restores the PAN ID and network key, but the EFR32's smaller link-key table means individual device keys often don't transfer — devices will rejoin with a button press, keeping their friendly names.

The approach uses [`zigpy-cli`](https://github.com/zigpy/zigpy-cli) to clone network credentials (PAN ID, network key, channel) from the old coordinator to the new one. The credential-cloning method works for other coordinator pairs that `zigpy-cli` supports, but the specific commands, config values, and HA steps in this guide are written for this exact migration path.

---

## Prerequisites

- Home Assistant OS with the Zigbee2MQTT add-on installed
- **Advanced SSH & Terminal add-on installed and running** — this is required to run `zigpy-cli` commands on the HA host. Install it from the Add-on Store (Settings → Add-ons → Add-on Store → search "Advanced SSH & Terminal"). The standard Terminal add-on is not sufficient.
- **[Spook](https://spook.boo) installed** — recommended. Spook's `entity: replace` service makes it straightforward to rename entity IDs that change after the migration, preserving history and updating all references automatically.
- [SLZB-06](https://amzn.to/4ejFiYz) still connected and Z2M running
- [ZBT-2](https://ameridroid.com/products/home-assistant-connect-zbt-2) physically connected via USB to the HA host

---

## Phase 1: Pre-Migration Preparation

### 1.1 Record Your Network Credentials

Before touching anything, note your current network settings from Z2M's `configuration.yaml` or the Z2M frontend. You'll need these to verify the restore succeeded and to set explicit values in the config afterward.

```
Channel: (e.g. 15)
PAN ID: (e.g. 0x3AD6)
Extended PAN ID: (e.g. 0e:89:33:d2:f8:e8:74:5f)
Network Key: (16-byte key)
```

> [!note] Extended PAN ID format
> Z2M displays the Extended PAN ID as colon-separated hex (e.g. `0e:89:33:d2:f8:e8:74:5f`). If you ever need to supply it in `configuration.yaml`, Z2M expects a decimal integer array: `[14, 137, 51, 210, 248, 232, 116, 95]`. Convert each hex byte to decimal.

### 1.2 Create Backups

Create three layers of backup before proceeding:

**1. HA add-on snapshot**  
Settings → System → Backups → Create backup. This captures `configuration.yaml`, `coordinator_backup.json`, and `database.db`.

**2. zigpy-cli radio backup**  
Install `zigpy-cli` in the HA Advanced SSH terminal (Alpine Linux — the TubesZB add-on may not work on recent HA versions):

```sh
apk add python3 py3-pip py3-virtualenv
python3 -m venv /config/zigpy-env
source /config/zigpy-env/bin/activate        # bash/sh
# fish shell: source /config/zigpy-env/bin/activate.fish
pip install zigpy-cli
```

The venv lives in `/config/` so it persists across restarts.

Stop Z2M, then run the backup. Replace the port/address with your old coordinator's address:

```sh
# Network-attached coordinator (e.g. SLZB-06 — https://amzn.to/4ejFiYz)
zigpy -v radio znp socket://<HOST>:6638 backup coordinator-backup.json

# USB coordinator
zigpy -v radio znp /dev/ttyUSB0 backup coordinator-backup.json
```

**3. Device reference document**  
List all devices with their IEEE addresses, Z2M friendly names, and HA entity IDs. If the migration requires full re-pairing, this checklist is essential. Export from the Z2M frontend or query `database.db` directly (see [Notes on database.db](#notes-on-databasedb)).

### 1.3 Optional: Pre-Migration LQI Baseline

Run a Z2M network map scan to force devices to report. Extract per-device LQI values as a baseline to compare post-migration signal quality. Flag any devices with median LQI ≤ 50 — these are weak-signal candidates that may drop off first.

### 1.4 Optional: Channel Energy Scan

Run a channel energy scan to check for interference. Useful if you're considering a channel change during migration. Zigbee channels 23–25 overlap with WiFi channel 11 (2.462 GHz) — avoid these if your environment has a router on channel 11.

```sh
zigpy -v radio znp tcp://<HOST>:<PORT> energy-scan
```

---

## Phase 2: Flash New Coordinator Firmware

> [!warning]
> If your new coordinator (e.g. [ZBT-2](https://ameridroid.com/products/home-assistant-connect-zbt-2)) arrived without Zigbee firmware loaded, `zigpy-cli` will return a generic `TimeoutError` with no explanation. **Flash firmware first.**

For the [ZBT-2](https://ameridroid.com/products/home-assistant-connect-zbt-2) on HA OS: Settings → Hardware → ZBT-2 → Update firmware.

> [!warning] Pick the correct firmware build
> The update screen offers two options: **Zigbee (EmberZNet NCP)** and Thread (OpenThread RCP). You must select the **Zigbee NCP** build (7.4.x or newer). If you flash the Thread firmware, `zigpy-cli` will throw the same generic `TimeoutError` — the symptom looks identical to a missing-firmware error but the fix is to reflash with the correct build.

For other coordinators, follow the manufacturer's flashing instructions before proceeding.

---

## Phase 3: Transfer and Restore

### 3.1 Transfer the Backup File to the HA Host

If you created the backup on a remote machine (e.g. your desktop), transfer it to the HA host. Note: HA's SSH server doesn't support the SCP subsystem — use `rsync` instead. Transfer to `/config/` (persistent), not `/root/` (ephemeral). The Advanced SSH add-on runs on port 22222:

```sh
rsync -av -e "ssh -p 22222" coordinator-backup.json root@homeassistant.local:/config/
```

Alternatively, use Samba or the File Editor add-on to drop the file into `/config/`.

### 3.2 Restore to the New Coordinator

Run from the HA Advanced SSH terminal with the venv active:

```sh
source /config/zigpy-env/bin/activate.fish

# EZSP/EmberZNet coordinator (e.g. ZBT-2 — https://ameridroid.com/products/home-assistant-connect-zbt-2)
zigpy -v radio --baudrate 460800 ezsp /dev/ttyACM0 restore /config/coordinator-backup.json
```

> [!warning] zigpy-cli argument order matters
> `--baudrate` must come **before** the radio type, not after. This fails:
> ```sh
> zigpy -v radio ezsp /dev/ttyACM0 --baudrate 460800 restore  # WRONG
> ```
> This works:
> ```sh
> zigpy -v radio --baudrate 460800 ezsp /dev/ttyACM0 restore  # CORRECT
> ```

> [!note]
> The `--flow-control` flag does not exist in recent versions of zigpy-cli. Remove it if you see it in older guides.

**Expected output:** Many `sl_Status.FAIL` warnings for individual device key entries. These are **normal and harmless** — the EFR32 chip has a smaller key table than ZNP, so entries that don't fit are silently skipped. Devices whose keys didn't transfer will need to rejoin manually, but the network credentials are fully restored.

### 3.3 Verify the IEEE Address

The coordinator's IEEE address must match the old [SLZB-06](https://amzn.to/4ejFiYz) — Z2M uses it to identify the coordinator. `zigpy-cli restore` writes the IEEE from the backup, but on Ember it can fail silently.

After the restore, verify:

```sh
zigpy -v radio --baudrate 460800 ezsp /dev/ttyACM0 info
```

Check that the IEEE in the output matches your old coordinator's IEEE. If it doesn't, write it explicitly:

```sh
zigpy -v radio --baudrate 460800 ezsp /dev/ttyACM0 write-ieee <old_ieee>
```

### 3.4 Verify the Restore

Power-cycle the new coordinator (unplug, wait 10 seconds, replug).

When you start Z2M, check the logs immediately:

- ✅ **Success:** `zigbee-herdsman started (resumed)` — existing network found
- ❌ **Failure:** `forming new network` — stop Z2M immediately and retry from Phase 3

---

## Phase 4: Update Z2M Configuration

Edit `configuration.yaml` to point at the new coordinator.

**Old config (example — ZNP/network-attached):**
```yaml
serial:
  port: tcp://192.168.x.x:6638
  baudrate: 115200
  adapter: zstack
  disable_led: false
```

**New config (example — EZSP/USB [ZBT-2](https://ameridroid.com/products/home-assistant-connect-zbt-2)):**
```yaml
serial:
  port: /dev/serial/by-id/usb-Nabu_Casa_ZBT-2_XXXXXXXX-if00
  baudrate: 460800
  adapter: ember
  rtscts: true
```

Notes:
- `disable_led` is ZNP-only — remove it for EZSP adapters
- `rtscts: true` enables hardware flow control, required by the [ZBT-2](https://ameridroid.com/products/home-assistant-connect-zbt-2) at 460800 baud
- Use the stable `/dev/serial/by-id/` path — `/dev/ttyACM0` changes on every USB re-plug or reboot. The by-id path is mapped into the Z2M container when you select it in Settings → Add-ons → Zigbee2MQTT → Configuration → Serial device. Replace `XXXXXXXX` with your device's actual ID (find it by running `ls /dev/serial/by-id/` in the SSH terminal)

**Do not pre-populate the `advanced:` section** unless Z2M complains. After a successful restore, Z2M reads the PAN ID, network key, and channel directly from the coordinator — adding them manually can cause a "configuration not consistent with state/backup" error if the format doesn't match exactly.

If Z2M logs `"Current backup file is not for EmberZNet stack"` or similar, rename `coordinator_backup.json` first (see below), then add explicit credentials using **decimal integers** — not hex strings:

```yaml
advanced:
  pan_id: 15062              # integer, not "0x3AD6"
  ext_pan_id: [14, 137, 51, 210, 248, 232, 116, 95]   # decimal bytes
  network_key: [##, ##, ##, ##, ##, ##, ##, ##, ##, ##, ##, ##, ##, ##, ##, ##]
  channel: 15
```

**Rename the old coordinator backup** to prevent a credential mismatch error on startup:
```sh
mv /config/coordinator_backup.json /config/coordinator_backup.json.bak
```

> [!warning]
> If you edit `configuration.yaml` via a mounted network share from another machine, verify the file was actually written before restarting Z2M. Edits made via a mounted volume may not persist if the share is stale.

### Map the Device in HA Add-on Settings

The Z2M container needs explicit permission to access the USB device:

Settings → Add-ons → Zigbee2MQTT → Configuration → Serial device → select your new coordinator's device path.

Without this, the container can't see `/dev/ttyACM0` even though it exists on the host.

---

## Phase 5: Device Rejoining

Despite correct credential transfer, many devices — especially routers — may not automatically rejoin. The EFR32 key table limitation means individual device link keys often don't transfer, so devices need to re-authenticate.

This does **not** require removing and re-adding devices from scratch. Simply trigger the normal pairing/join process on each device (usually a button press or power-cycle), and the device will rejoin and reappear in Z2M with its existing friendly name and configuration preserved.

**Rejoin routers first.** Router devices rebuild the mesh, which allows end devices to rejoin through them. Power-cycling mains-powered routers (outlets, smart switches, plug-in bulbs) often triggers an automatic rejoin. For devices with a pairing button, a short press is usually sufficient.

When opening the network for rejoining, **permit join via coordinator only** for the first 2–3 routers rather than opening the entire network. Once those routers are online and the mesh is starting to form, switch to network-wide permit join for remaining devices. This avoids the "pairing new devices is not working" issue that can occur when the mesh is too sparse to route join requests.

Wait 10–15 minutes after handling routers before triggering end device rejoins — the mesh needs to stabilize first.

> [!note] Z2M "router" classification vs. Zigbee spec
> Some mains-powered devices are classified as routers by Z2M but are actually `EndDevice` in the Zigbee spec — they don't extend the mesh. To check, read `database.db` directly (see [Notes on database.db](#notes-on-databasedb)) and look at the `"type"` field per device.

### Entity ID Changes After Rejoining

In most cases, devices rejoin with their existing friendly names and HA entity IDs intact. Occasionally a device re-registers with a slightly different entity ID (e.g. a suffix gets added or changed). If this happens:

1. Update automations and dashboards to reference the new entity ID
2. Delete the old stale entity: Settings → Devices & Services → Entities

### Zigbee Groups

Zigbee groups are defined in `database.db`, which is not touched during migration. Z2M re-pushes the group table to the coordinator on startup, so groups should reappear automatically — you do not need to recreate them manually.

What *is* reset is the coordinator's internal group table. Z2M repopulates it from `database.db` when it starts, so as long as Z2M starts cleanly on the new coordinator, groups will be restored. If a group is missing, manually recreate it in the Z2M frontend using the exact same `friendly_name` as before.

---

## Phase 6: Post-Migration Cleanup

### Audit Stale Entity References

Use [Watchman](https://github.com/dummylabs/thewatchman) or search your `automations.yaml` and `lovelace.yaml` for old entity IDs that no longer exist. Common causes:

- Device rejoined with a slightly different entity ID
- Motion sensor re-registered with `_occupancy` suffix added or removed

### LQI Comparison

Run another network map scan and compare LQI values against your pre-migration baseline.

> [!note]
> ZNP (Z-Stack) and EZSP (EmberZNet) stacks report LQI using different calibration curves. Absolute values are **not directly comparable** across the migration. Use the change relative to each device's own baseline, not absolute values.

---

## Troubleshooting

| Symptom | Resolution |
|---------|------------|
| `TimeoutError` on restore, new coordinator | Flash Zigbee firmware first |
| `scp` to HA host fails ("subsystem request failed") | Use `rsync -av` instead |
| `--flow-control` flag not recognized | Remove it — not in recent zigpy-cli |
| `--baudrate` argument error | Place before radio type: `zigpy -v radio --baudrate 460800 ezsp PORT` |
| Fish shell: `source activate` fails with "case builtin" error | Use `source activate.fish` |
| Config edit via network share didn't save | Confirm write, then restart Z2M |
| Z2M can't open serial port | Map device in HA add-on settings UI (Settings → Add-ons → Z2M → Configuration → Serial device) |
| `/dev/serial/by-id/` path not found in SSH terminal | Run `ls /dev/serial/by-id/` — path appears once device is mapped in add-on settings |
| `sl_Status.FAIL` warnings during restore | Expected — EFR32 key table smaller than ZNP; harmless |
| IEEE address mismatch after restore | Run `zigpy info` to check; use `write-ieee <old_ieee>` to correct it |
| Z2M log says "forming new network" | Stop Z2M immediately; restore failed — retry Phase 3 |
| "configuration not consistent with state/backup" | Add explicit `pan_id`/`ext_pan_id`/`network_key`/`channel` in config; rename `coordinator_backup.json` to `.bak` |
| "Table full" during restore | Full power-cycle new coordinator, retry restore |
| Router devices didn't auto-rejoin | Trigger rejoin via pairing button or power-cycle; start with routers to rebuild mesh |
| Devices got new HA entity IDs | Update automations/dashboards; delete stale entities |
| Zigbee groups missing after migration | Z2M should re-push from `database.db` on startup; if not, recreate manually in Z2M frontend with exact same `friendly_name` |

---

## Caveats

**Automatic rejoin is not guaranteed.** The credential transfer preserves the network identity (PAN ID, network key, channel), but individual device link keys may not transfer due to the EFR32 chip's smaller key table. Devices without a transferred key won't rejoin automatically. In practice, expect to manually trigger a rejoin on most router devices and some end devices — this is just a button press or power-cycle. Devices reappear in Z2M with their existing friendly names and configuration intact; `database.db` is untouched.

**The network will be disrupted.** There is downtime between stopping Z2M on the old coordinator and confirming Z2M is running on the new one. Plan accordingly for time-sensitive automations.

---

## Notes on `database.db`

Z2M's `database.db` is **newline-delimited JSON**, not SQLite, despite the `.db` extension. Don't use `sqlite3` to query it. Use Python or any JSON-lines parser:

```python
import json

with open("database.db") as f:
    devices = [json.loads(line) for line in f if line.strip()]

# Find actual Zigbee router devices (not just Z2M-classified routers)
routers = [d for d in devices if d.get("type") == "Router"]
```

---

## References

- [Z2M: How to migrate from one adapter to another](https://www.zigbee2mqtt.io/guide/faq/#how-do-i-migrate-from-one-adapter-to-another)
- [Z2M: What does and does not require re-pairing of all devices](https://www.zigbee2mqtt.io/guide/faq/#what-does-and-does-not-require-re-pairing-of-all-devices)
- [Z2M community: zigpy migration method](https://github.com/Koenkk/zigbee2mqtt/discussions/26716)
- [zigpy-cli](https://github.com/zigpy/zigpy-cli)
- [Watchman for HA](https://github.com/dummylabs/thewatchman)
- [Spook — HA integration](https://spook.boo)
- [Home Assistant Connect ZBT-2](https://ameridroid.com/products/home-assistant-connect-zbt-2)
- [SLZB-06](https://amzn.to/4ejFiYz)
