# Home Assistant + Zigbee (YouTube Tutorial)

Docker-based Home Assistant setup with a USB Zigbee coordinator via the ZHA integration. Companion repo for the YouTube Zigbee tutorial.

## What this gives you

- Home Assistant (`ghcr.io/home-assistant/home-assistant:stable`) on Docker
- Web UI on port 80 (mapped to container's 8123)
- USB Zigbee coordinator passed through (`/dev/ttyUSB0`)
- ZHA debug logging pre-configured

## Requirements

- Linux host with Docker + Docker Compose
- USB Zigbee coordinator (e.g. Sonoff ZBDongle-E/P, SkyConnect, ConBee II — any ZHA-supported stick)
- A Zigbee end device to pair (bulb, sensor, switch, etc.)

## Quick start

1. Plug in the coordinator and find its device path:

   ```bash
   ls -l /dev/serial/by-id/
   ```

   Use the stable `/dev/serial/by-id/...` path, not bare `/dev/ttyUSB0` (enumeration order changes across reboots).

2. Update `docker-compose.yml` with your actual device ID:

   ```yaml
   devices:
     - /dev/ttyUSB0:/dev/ttyUSB0
     - /dev/serial/by-id/<device_id>:/dev/serial/by-id/<device_id>
   ```

3. Start Home Assistant:

   ```bash
   docker compose up -d
   docker compose logs -f
   ```

4. Open `http://<host-ip>` and finish onboarding.

5. Add ZHA: **Settings → Devices & Services → Add Integration → Zigbee Home Automation**, select your serial port. Leave radio type on auto-detect unless you know your chip (EZSP/EmberZNet vs. ZNP vs. deCONZ).

6. Put your Zigbee device in pairing mode, then in ZHA click **Add Device**.

## Configuration

`docker-compose.yml` is the only tracked config. Runtime state lives in `./config/` (gitignored) — includes `configuration.yaml`, `automations.yaml`, `.storage/`, logs, and the DB.

Key compose details:

| Setting | Why |
|---|---|
| `ports: "80:8123"` | Serve UI on port 80 |
| `/run/dbus` and `/run/udev` mounts | USB discovery / hardware info |
| `devices` | Coordinator passthrough |
| `NET_ADMIN`, `NET_RAW` | mDNS/discovery for HA |
| `TZ=${TZ:-Etc/UTC}` | Override with `TZ=Asia/Colombo docker compose up -d` etc. |

## Debug logging (ZHA / zigpy)

If pairing fails or the coordinator won't connect, enable verbose Zigbee logs in `config/configuration.yaml`:

```yaml
logger:
  default: info
  logs:
    homeassistant.components.zha: debug
    zigpy: debug
```

Then restart and watch:

```bash
docker compose restart
docker compose logs -f | grep -i -E "zha|zigpy|bellows"
# or inside the container: tail -f /config/home-assistant.log
```

Common things to look for: wrong serial path, wrong baudrate/radio type, or the stick being claimed by another driver/container (e.g. ModemManager holding `/dev/ttyUSB0` — disable or add a udev rule).

## Troubleshooting

- **No `/dev/ttyUSB0` in container:** check the host path exists, fix the `devices:` mapping, `docker compose up -d --force-recreate`.
- **ZHA probe failures (`Failed to probe with config ...`):** wrong radio type/baudrate, or the stick is in use elsewhere.
- **Device won't pair:** bring it within a couple meters of the coordinator, factory-reset it, and retry discovery. USB 3.0 interference is real — use a USB 2.0 port or extension cable.

## Repo layout

```
.
├── docker-compose.yml   # HA service definition (tracked)
├── README.md
└── config/              # HA runtime data (gitignored, created on first run)
```
