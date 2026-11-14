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

<table>
  <tr>
    <td align="center"><img src="docs/img/1-attesa.webp" width="150"></td>
    <td align="center"><img src="docs/img/2-carica.webp" width="150"></td>
    <td align="center"><img src="docs/img/3-stop.webp" width="150"></td>
    <td align="center"><img src="docs/img/4-cento.webp" width="150"></td>
    <td align="center"><img src="docs/img/5-log.webp" width="150"></td>
  </tr>
</table>

## 🚀 Installation

```bash
git clone https://github.com/SiNaPsEr0x/ricarica-ia.git
cd ricarica-ia
cp .env.example .env      # set SAIC_USER and SAIC_PASSWORD (secondary MG account)
docker compose up -d
```

With Portainer: **Stacks → Add stack**, paste [`docker-compose.yml`](docker-compose.yml) and add `SAIC_USER` and `SAIC_PASSWORD` as environment variables.

Then open `http://RASPBERRY_IP:8088` from your phone.

## 🔧 Main settings

In the `ev-limiter` service of [`docker-compose.yml`](docker-compose.yml):

| Variable | Default | What it does |
|---|---|---|
| `PRISM_TOPIC` | `home/prism/` | MQTT base topic, same as in the Silla app |
| `PRISM_PORT` | `1` | Wallbox socket (1 or 2) |
| `BATTERY_KWH` | `51` | Usable battery capacity |
| `TARGET_SOC` | `80` | Initial limit (then change it from the web page) |
| `AUTO_ARM` | `true` | Arms itself when the cable is plugged in |

## 📡 Setting up the Prism

In the **Silla** app: Settings → Access and communication → **Custom MQTT**

- **Host:** Raspberry IP (not `127.0.0.1`)
- **Port:** `1883`
- **Base topic:** `home/prism/`
- **TLS:** OFF · **User/Password:** anything

Check: on the web page, State / Mode / Power must show values.

## 🕹️ How to use it

- **Avvia** (Start): reads the level and starts charging towards the target
- **Stop**: pause and disarm
- **Disarma** (Disarm): removes the limit, charging continues freely
- **100% solo questa volta** (100% just this once): free charge, then back to the usual target
- **− / +**: change the target (20–100%)
- **Log**: real-time journal

## ⚠️ Warning

- The web page (port 8088) and the MQTT broker (1883) **have no password**: keep them **on the local network only**, never on the Internet.
- If the car doesn't answer, charging stays paused: press **Disarma** (Disarm) to force it.
- Tested only with MG4 Standard + Silla Prism Solar.

## 📄 License

MIT, use at your own risk: check that charging stops as expected. Independent project, **not affiliated** with MG, SAIC, Silla or Raspberry Pi. Built on [SAIC Python MQTT Gateway](https://github.com/SAIC-iSmart-API/saic-python-mqtt-gateway) and [Eclipse Mosquitto](https://mosquitto.org/).
