# 🌍 Netatmo → TRMNL Dashboard Bridge 🚀

This project runs on a Raspberry Pi 🥧 and acts as a bridge between a Netatmo Healthy Home Coach (NHC) and a [TRMNL](https://usetrmnl.com) display.  

It provides:
- 🔌 A local Bottle web service (`/metrics`) that fetches Netatmo metrics (CO₂, Temperature, Humidity, Noise, Pressure).  
- ⏳ Caching and token refresh logic to respect Netatmo’s API limits (24 refreshes per day, access token valid ~3 hours).  
- 📡 A JSON endpoint TRMNL can poll every 5 minutes.  
- 🌐 Optional DuckDNS + port forwarding for external access.  
- 📊 TRMNL plugin markup for a clean 2×2 quadrant dashboard with a footer.

<img width="675" height="404" alt="image" src="https://github.com/user-attachments/assets/cd90fa92-3d38-4151-a4d3-5d3106a88887" />


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

Clone or create a project folder:
```bash
mkdir ~/netatmo-trmnl && cd ~/netatmo-trmnl
```

---

## 3. Environment file 🔑

Create `netatmo.env` in the project folder:

```ini
NETATMO_CLIENT_ID=your_client_id
NETATMO_CLIENT_SECRET=your_client_secret
NETATMO_REFRESH_TOKEN=your_refresh_token
# Optional: pin to a device
NETATMO_DEVICE_ID=70:ee:50:af:af:90
PORT=8000
```

---

## 4. Python dependencies 📦

```bash
pip3 install bottle requests python-dotenv
```

---

## 5. The Bottle App (`app.py`) 🐍

```python
# Full code from app.py here (same as provided in chat)
```

---

## 6. Run as a service ⚙️

Create `netatmo.service`:

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

Enable it:
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

Expected output:
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
<!-- Full TRMNL HTML markup goes here (same as provided in chat) -->
```

---

## 9. Notes on Rate Limits ⏱️
- Netatmo **access tokens** are valid ~3 hours.  
- Netatmo **refresh tokens** can only be exchanged **24 times per day per user/app**.  
- This service refreshes once every ~3 hours, staying safe under the limit.  
- TRMNL polls every 5 minutes but the Pi caches Netatmo data for 5 minutes to avoid extra API calls.  

---

## ⚠️ Disclaimer
This README was generated with the assistance of AI 🤖. Please review and adapt the instructions for your own environment, security requirements, and version of software.  
