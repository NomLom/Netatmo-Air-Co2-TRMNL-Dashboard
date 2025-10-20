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

Full source provided in `app.py` above.

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
- Polling URL: `http://slidek9.duckdns.org:8000/metrics`  
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
