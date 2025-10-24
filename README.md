# 🌍 Netatmo → TRMNL Dashboard Bridge 🚀

This project runs on a Raspberry Pi 🥧 and acts as a bridge between a Netatmo Healthy Home Coach (NHC) and a [TRMNL](https://usetrmnl.com) display.  

It provides:
- 🔌 A local Bottle web service (`/metrics`) that fetches Netatmo metrics (CO₂, Temperature, Humidity, Noise, Pressure).  
- ⏳ Caching and token refresh logic to respect Netatmo’s API limits (24 refreshes per day, access token valid ~3 hours).  
- 📡 A JSON endpoint TRMNL can poll every 5 minutes.  
- 🌐 Optional DuckDNS + port forwarding for external access.  
- 📊 TRMNL plugin markup for a clean 2×2 quadrant dashboard with a footer.

<img width="675" height="404" alt="image" src="https://github.com/user-attachments/assets/c90dc7cb-ccd0-4cea-b2e9-ec4e9d65e6f8" />


---

## 1. Prerequisites ✅
- Raspberry Pi running Raspberry Pi OS (Lite or Desktop).  
- Python 3.7+ installed.  
- A Netatmo Developer App (client ID, secret, refresh token).  
- DuckDNS domain and router port forwarding if you want external access.  

---

## 2. Installation on Raspberry Pi 🛠️

```bash
sudo apt-get update
sudo apt-get install -y python3 python3-pip python3-venv
```

Create project folder:
```bash
mkdir ~/netatmo-trmnl && cd ~/netatmo-trmnl
```

---

## 3. Environment file 🔑

Create `netatmo.env`:

```ini
NETATMO_CLIENT_ID=your_client_id
NETATMO_CLIENT_SECRET=your_client_secret
NETATMO_REFRESH_TOKEN=your_refresh_token
PORT=8000
# Optional device id:
# NETATMO_DEVICE_ID=70:ee:50:af:af:90
```

---

## 4. Python dependencies 📦

```bash
pip3 install bottle requests python-dotenv
```

---

## 5. The Bottle App (`app.py`) 🐍

```
import json, os, time
from pathlib import Path
from datetime import datetime, timezone
import requests
from dotenv import load_dotenv
from bottle import Bottle, response, run

load_dotenv("netatmo.env")

CLIENT_ID     = os.environ["NETATMO_CLIENT_ID"]
CLIENT_SECRET = os.environ["NETATMO_CLIENT_SECRET"]
REFRESH_TOKEN = os.environ["NETATMO_REFRESH_TOKEN"]
DEVICE_ID     = os.environ.get("NETATMO_DEVICE_ID") or None
PORT          = int(os.environ.get("PORT", "8000"))

CACHE_PATH    = Path("netatmo_cache.json")
DATA_TTL      = 300   # 5 minutes
EARLY_REFRESH = 120   # refresh 2 min early

app = Bottle()

def _now_ms(): return int(time.time() * 1000)

def load_cache():
    if CACHE_PATH.exists():
        try: return json.loads(CACHE_PATH.read_text())
        except: pass
    return {"access_token": None, "refresh_token": REFRESH_TOKEN, "token_expires_at": 0, "latest_data": None, "latest_fetched_at": 0}

def save_cache(c): CACHE_PATH.write_text(json.dumps(c, indent=2))

def refresh_access_token(cache):
    url = "https://api.netatmo.com/oauth2/token"
    data = {
        "grant_type": "refresh_token",
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET,
        "refresh_token": cache.get("refresh_token") or REFRESH_TOKEN
    }
    r = requests.post(url, data=data, timeout=30)
    r.raise_for_status()
    body = r.json()
    cache["access_token"] = body["access_token"]
    cache["refresh_token"] = body.get("refresh_token", cache.get("refresh_token"))
    cache["token_expires_at"] = _now_ms() + int(body.get("expires_in", 10800)) * 1000
    save_cache(cache)

def need_refresh(cache):
    return (not cache.get("access_token")) or (_now_ms() > (cache.get("token_expires_at", 0) - EARLY_REFRESH*1000))

def fetch_homecoach(access_token):
    url = "https://api.netatmo.com/api/gethomecoachsdata"
    headers = {"Authorization": f"Bearer {access_token}", "User-Agent": "netatmo-pi/1"}
    r = requests.get(url, headers=headers, timeout=30)
    if r.status_code == 401: raise PermissionError("unauthorized")
    r.raise_for_status()
    devices = r.json()["body"].get("devices", [])
    if not devices: raise RuntimeError("No devices found")
    device = devices[0]
    if DEVICE_ID:
        device = next((d for d in devices if d.get("_id") == DEVICE_ID), device)
    dash = device.get("dashboard_data", {})
    ts = int(dash.get("time_utc", 0))
    iso = datetime.fromtimestamp(ts, tz=timezone.utc).isoformat()
    return {
        "success": True,
        "temperature_c": dash.get("Temperature"),
        "co2_ppm": dash.get("CO2"),
        "humidity_pct": dash.get("Humidity"),
        "noise_db": dash.get("Noise"),
        "pressure_hpa": dash.get("Pressure"),
        "time_utc": ts,
        "ts": iso,
        "device_id": device.get("_id"),
        "station_name": device.get("station_name", "")
    }

@app.route("/metrics")
def metrics():
    cache = load_cache()
    if need_refresh(cache):
        try: refresh_access_token(cache)
        except Exception as e:
            if cache.get("latest_data"):
                return {**cache["latest_data"], "stale": True, "error": str(e)}
            response.status = 503; return {"success": False, "error": str(e)}

    fresh_enough = (_now_ms() - cache.get("latest_fetched_at", 0)) < (DATA_TTL * 1000)
    if not fresh_enough:
        data = fetch_homecoach(cache["access_token"])
        cache["latest_data"] = data
        cache["latest_fetched_at"] = _now_ms()
        save_cache(cache)

    response.content_type = "application/json"
    return json.dumps(cache["latest_data"])

if __name__ == "__main__":
    run(app, host="0.0.0.0", port=PORT)
```

---

## 6. Run as a service ⚙️

Create `/etc/systemd/system/netatmo.service`:

```ini
[Unit]
Description=Netatmo → TRMNL bridge (Bottle)
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/netatmo-trmnl
ExecStart=/usr/bin/python3 /home/pi/netatmo-trmnl/app.py
EnvironmentFile=/home/pi/netatmo-trmnl/netatmo.env
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now netatmo.service
```

Check logs:
```bash
journalctl -u netatmo.service -f
```

---

## 7. Test 🔍

```bash
curl http://192.168.0.199:8000/metrics
```

Example output:
```json
{
  "success": true,
  "temperature_c": 21.1,
  "co2_ppm": 808,
  "humidity_pct": 60,
  "noise_db": 51,
  "pressure_hpa": 993.4,
  "time_utc": 1760900442,
  "ts": "2025-10-20T19:15:42Z",
  "device_id": "70:ee:50:af:af:90",
  "station_name": "Smol Rom"
}
```

---

## 8. TRMNL Plugin Setup 🖥️

- Strategy: **Polling**  
- Polling URL: `<your dynampic IP>  
- Verb: `GET`  
- Interval: **5 minutes**  
- Remove bleed margin: **Yes**  

### Markup (TRMNL dashboard)

```html
<div class="layout layout--col gap--none" style="height:100%; color:black;">

  <div class="title_bar bg--gray-75 border--h-6 rounded--none" style="color:black;">
    <span class="title" style="color:black;">Netatmo</span>
    <span class="instance" style="color:black;">Last updated: {{ ts | default: time_utc | date: "%Y-%m-%d %H:%M" }}</span>
  </div>

  <div style="display:flex; flex-direction:column; flex:1 1 auto; min-height:0;">
    <div class="grid grid--cols-2 gap--none"
         style="flex:1 1 auto; min-height:0; grid-template-rows: 1fr 1fr; grid-auto-rows: minmax(0,1fr);">

      <div class="item {% if co2_ppm > 1100 %}bg--gray-60{% endif %} border--h-4 border--v-4"
           style="display:flex; flex-direction:column; justify-content:center; align-items:center;">
        <span class="value value--tnums value--xxxlarge" data-value-fit="true">{{ co2_ppm }}</span>
        <span class="label">ppm CO₂</span>
      </div>

      <div class="item border--h-4 border--v-4"
           style="display:flex; flex-direction:column; justify-content:center; align-items:center;">
        <span class="value value--tnums value--xxxlarge" data-value-fit="true">{{ temperature_c }}</span>
        <span class="label">°C Temperature</span>
      </div>

      <div class="item border--h-4 border--v-4"
           style="display:flex; flex-direction:column; justify-content:center; align-items:center;">
        <span class="value value--tnums value--xxxlarge" data-value-fit="true">{{ noise_db }}</span>
        <span class="label">dB Noise</span>
      </div>

      <div class="item border--h-4 border--v-4"
           style="display:flex; flex-direction:column; justify-content:center; align-items:center;">
        <span class="value value--tnums value--xxxlarge" data-value-fit="true">{{ humidity_pct }}</span>
        <span class="label">% Humidity</span>
      </div>
    </div>

    <div class="item {% if pressure_hpa < 995 %}bg--gray-70{% endif %} border--h-6 rounded--none"
         style="display:flex; flex-direction:column; justify-content:center; align-items:center;">
      <span class="value value--tnums value--xlarge">{{ pressure_hpa }}</span>
      <span class="label">hPa Pressure{% if pressure_hpa < 995 %} — Low{% endif %}</span>
    </div>
  </div>
</div>
```

---

## 9. Notes on Rate Limits ⏱️

- Netatmo **access tokens** are valid ~3 hours.  
- Netatmo **refresh tokens** can only be exchanged **24 times per day per user/app**.  
- This service refreshes once every ~3 hours, staying safe under the limit.  
- TRMNL polls every 15 minutes but the Pi caches Netatmo data for 5 minutes to avoid extra API calls.  

---

## ⚠️ Disclaimer

This README was generated with the assistance of AI 🤖. Please review and adapt the instructions for your own environment, security requirements, and version of software.  
