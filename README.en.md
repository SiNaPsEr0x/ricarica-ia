<div align="center">

[**🇮🇹 Italiano**](README.md) &nbsp;|&nbsp; **🇬🇧 English**

<img src="docs/img/hero.webp" alt="Ricarica IA" width="100%">

# ⚡ Ricarica IA

### Charging that **stops by itself**.

Plug in the cable and it checks the battery, then stops charging at the level you choose (default 80%).

![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![MQTT](https://img.shields.io/badge/MQTT-Mosquitto-660066?logo=eclipsemosquitto&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-a78bfa)

**[🌐 Presentation](https://sinapser0x.github.io/ricarica-ia/)** *(Italian)*

</div>

> ⚠️ **The software is currently Italian-only** (web interface, screenshots, presentation). The documentation is bilingual. Italian button names are given in parentheses below.

## 🔌 What you need

Tested **only** with this combination:

- **Wallbox:** Silla Prism Solar (with custom MQTT)
- **Car:** MG4 Standard, with the MG iSMART app
- **Raspberry Pi** (or any always-on mini-server) with Docker/Portainer, on the same network as the Prism
- A **secondary MG iSMART account**, added permanently as a family member of the car (don't use your main one)

## 🧭 How it works

1. You plug in the cable → charging is paused immediately.
2. It reads the **real battery level** from the MG cloud (retries if the car doesn't answer).
3. Charging starts and is **supervised** (car data + backup estimate from the kWh delivered).
4. At the target → **automatic pause**.

Only the battery reading goes through the cloud, and only while the cable is plugged in. Wallbox commands stay on your local network.

### 🛑 How the stop is decided

- **Backup estimate** = starting level + kWh delivered by the Prism × efficiency (`CHARGE_EFFICIENCY`, 0.92) ÷ capacity. It is recalibrated on every fresh cloud reading.
- It stops when the **cloud reading** (no older than 10 min) reaches the target. If the cloud doesn't answer or is stale, the **estimate** decides.
- If the cloud lags but the estimate exceeds the target by `EST_MARGIN` (1 point), the estimate wins ("safety estimate").
- **Maximum theoretical overshoot: about `EST_MARGIN` percentage points.** At 2.8 kW the estimate advances about 0.2 points every 2 minutes, which is the refresh interval near the target.
- The pause command is repeated until the Prism confirms "Pause". Safety timeout: 12 hours.
- Cloud refreshes: every 2 min within 3 points of the target, every 10 min when further away; if the cloud doesn't answer the interval doubles (max 10 min).
- The remaining charging time reported by the car (computed for 100%) is **not** used.

### 🔁 Takeover (even after a power outage)

Ricarica IA takes command of the Prism **even when it is already delivering power**:

- If the Prism is charging and the limit is not armed (stack restart, blackout, charge started from the Silla app), it **pauses**, reads the level from the cloud and **re-arms** towards the target. If the level is already at the target, it stays paused.
- The session (armed / at target / stop / free) and "100% just this once" are **saved to a file** (`/deps/state.json`, `pydeps` volume) and survive restarts.
- Manual **Stop** and **Disarma** (Disarm) are respected after a restart too: while the cable stays plugged in it won't take control again. Unplugging the cable starts fresh.
- If the Prism itself reboots and detects the cable again, the session is treated as new and the limit is applied.
- Anti-loop: at least `TAKEOVER_COOLDOWN_SEC` seconds (120) between two takeovers.
- Mosquitto saves its data every 30 seconds (`autosave_interval`, default is 30 minutes): after a blackout the restored data is recent.
- If the cloud doesn't answer, charging stays paused until the level arrives: press **Disarma** (Disarm) to force it.
- To get the previous behaviour back (arm only when the cable is plugged in): `TAKEOVER=false`.

<table>
  <tr>
    <td align="center"><img src="docs/img/1-attesa.webp" width="150"></td>
    <td align="center"><img src="docs/img/2-carica.webp" width="150"></td>
    <td align="center"><img src="docs/img/3-stop.webp" width="150"></td>
    <td align="center"><img src="docs/img/4-cento.webp" width="150"></td>
    <td align="center"><img src="docs/img/5-log.webp" width="150"></td>
  </tr>
</table>

## 🚀 Installation with Portainer

1. **Stacks → Add stack** and give it a name (e.g. `ricarica-ia`).
2. Choose **Web editor** and paste the contents of [`docker-compose.yml`](docker-compose.yml).
3. Below the editor expand **Environment variables** and add two variables (one by one, or with **Load variables from .env file** starting from [`.env.example`](.env.example)):
   - `SAIC_USER` = email of the **secondary** MG iSMART account
   - `SAIC_PASSWORD` = its password
4. Click **Deploy the stack**.
5. Open `http://RASPBERRY_IP:8088` from your phone.

The compose reads the credentials with `${SAIC_USER}` and `${SAIC_PASSWORD}`: leave them like that in the YAML.

> 🔐 **Never write your email and password inside `docker-compose.yml`** and never commit your `.env` file. If the password contains `$`, it would have to be written `$$` in the YAML: with Portainer variables, check the log (see below).

**Check:** the `saic-gateway` container log must show `Logged in as …`.

### ✏️ Editing or updating the stack

**Stacks → your stack → Editor**: change the compose there (for example `TARGET_SOC` or `EST_MARGIN`) and expand **Environment variables** for the credentials. Then **Update the stack**; if you removed services, also tick **Prune services**. The limiter's state (session, target) lives in the `pydeps` volume.

> ⚠️ **Watchtower:** `ev-limiter` and `mosquitto` carry the label `com.centurylinklabs.watchtower.enable: "false"`. Watchtower recreates containers through the API without the `configs: content:` files (`app.py`, `mosquitto.conf`): the limiter would restart-loop with `can't open file '/app/app.py'` and no limit would be applied. Keep the label; to update images use **Update the stack** with **Re-pull image**.

*Git alternative:* **Add stack → Git Repository** with Compose path `docker-compose.yml` and the same variables. Git stacks have no Editor tab: to change settings you need your own fork of the repo.

*Without Portainer* (terminal): `git clone`, `cp .env.example .env` (fill in the two variables), then `docker compose up -d`.

## 🔧 Main settings

In the `ev-limiter` service of the stack (Portainer **Editor** tab, or [`docker-compose.yml`](docker-compose.yml)):

| Variable | Default | What it does |
|---|---|---|
| `PRISM_TOPIC` | `home/prism/` | MQTT base topic, same as in the Silla app |
| `PRISM_PORT` | `1` | Wallbox socket (1 or 2) |
| `BATTERY_KWH` | `51` | Usable battery capacity |
| `TARGET_SOC` | `80` | Initial limit (then change it from the web page) |
| `AUTO_ARM` | `true` | Arms itself when the cable is plugged in |
| `ACQUIRE_SEC` | `300` | How long it waits for the battery level on plug-in; then it stays paused and retries more slowly |
| `REFRESH_SEC` | `120` | Seconds between cloud requests within 3 points of the target |
| `REFRESH_FAR_SEC` | `600` | Same, when the target is further away |
| `CHARGE_EFFICIENCY` | `0.92` | Efficiency used by the estimate (energy into the battery ÷ energy from the Prism) |
| `EST_MARGIN` | `1.0` | Percentage points above the target after which the estimate stops charging anyway |
| `SOC_STALE_SEC` | `600` | Older than this, the cloud reading is no longer considered fresh |
| `GW_OFF` | `true` | Turns off the gateway's periodic polling: only Ricarica IA makes requests |
| `TAKEOVER` | `true` | Takes command of the Prism even when it is already charging without a limit (needs `AUTO_ARM`) |
| `TAKEOVER_COOLDOWN_SEC` | `120` | Minimum seconds between two takeovers |

In the `saic-gateway` service:

| Variable | Default | What it does |
|---|---|---|
| `MESSAGES_REQUEST_INTERVAL` | `900` | Seconds between message checks on the SAIC API (the gateway's own default is 60) |

## 📡 Setting up the Prism

In the **Silla** app: Settings → Access and communication → **Custom MQTT**

- **Host:** Raspberry IP (not `127.0.0.1`)
- **Port:** `1883`
- **Base topic:** `home/prism/`
- **TLS:** OFF · **User/Password:** anything

Check: on the web page, State / Mode / Power must show values.

## 🌍 Away from home with Tailscale

The Prism stays on the local network and points to the Raspberry's **LAN** IP. Away from home, your phone only needs the web page:

1. Install [Tailscale](https://tailscale.com/) on the Raspberry (on the host, not in a container) and on your phone.
2. Open `http://RASPBERRY-NAME:8088` (MagicDNS name) or `http://100.x.y.z:8088` (Tailscale IP).
3. Use Tailscale ACLs to allow only port `8088` from your own devices: `1883` is only needed by the Prism, on the LAN.

❌ **Don't use `tailscale funnel`** and don't forward ports on your router: Funnel exposes the service to the whole Internet, and the page has no password.

## 🕹️ How to use it

- The **all'80%** box (hours:min) shows the estimated time to the stop, computed from the Prism's power (`--:--` when not charging). The active button is highlighted.
- **Avvia** (Start): reads the level and starts charging towards the target
- **Stop**: pause and disarm
- **Disarma** (Disarm): removes the limit, charging continues freely
- **100% solo questa volta** (100% just this once): free charge, then back to the usual target
- **− / +**: change the target (20–100%)
- **Log**: real-time journal

## ⚠️ Warning

- The web page (port 8088) and the MQTT broker (1883) **have no password**: keep them **on the local network or Tailscale only**, never on the Internet.
- The `ev-limiter` service mounts the Docker socket (`docker.sock`) only to show the other containers' logs. Even read-only, it gives full Docker access to the host: if you don't need it, remove the `/var/run/docker.sock` line (the **GW** and **BRK** log tabs will stop working).
- If the car doesn't answer, charging stays paused: press **Disarma** (Disarm) to force it.
- Tested only with MG4 Standard + Silla Prism Solar.

## 📄 License

MIT, use at your own risk: check that charging stops as expected. Independent project, **not affiliated** with MG, SAIC, Silla or Raspberry Pi. Built on [SAIC Python MQTT Gateway](https://github.com/SAIC-iSmart-API/saic-python-mqtt-gateway) and [Eclipse Mosquitto](https://mosquitto.org/).
