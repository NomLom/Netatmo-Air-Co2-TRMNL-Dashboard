# TRMNL Netatmo Dashboard Plugin

This project sets up a **TRMNL private plugin** to display live metrics from a Netatmo Home Coach (Temperature, CO₂, Humidity, Noise, Pressure) in a 2×2 dashboard layout with a footer for pressure.
<img width="676" height="405" alt="image" src="https://github.com/user-attachments/assets/deba4d91-efeb-4d54-a4f5-2c811d5b8d83" />


---

## 🔑 Netatmo API Limits
- **24 calls per token per 24 hours (rolling window).**
- Effectively = **1 call/hour maximum** per `client_id + user_id` pair.
- Polling more frequently risks **temporary blocking** until under the quota.

To work around this, we use **Pipedream** as a proxy/cache:  
- Fetch from Netatmo only **once per hour**.  
- Cache the latest data.  
- TRMNL polls Pipedream as often as desired (5–10 min), but usually just gets the cached JSON.
- - I do't see the point in this bit if you pluging is configered to fetch once per hour btw.  

---

## ⚙️ Setup Steps

### 1. Netatmo App & Tokens
1. Create a Netatmo Developer App → get `client_id` and `client_secret` https://dev.netatmo.com/apps/.
2. Use OAuth2 to generate:
   - `access_token`
   - `refresh_token`
3. Required scope: `read_homecoach`.

### 2. Pipedream Workflow
We used a **HTTP trigger** workflow with these steps:

1. **HTTP Trigger** – entrypoint for TRMNL polling.
2. **Get Auth (HTTP Request)** – Refresh token call:
   ```http
   POST https://api.netatmo.com/oauth2/token
   Content-Type: application/x-www-form-urlencoded

   grant_type=refresh_token
   client_id=CLIENT_ID
   client_secret=CLIENT_SECRET
   refresh_token=REFRESH_TOKEN
   ```
   Returns new `access_token` (valid 3h).
   <img width="871" height="503" alt="image" src="https://github.com/user-attachments/assets/6265d5af-de5f-4359-95d5-d5881a551ca5" />

3. **Fetch Netatmo Data (HTTP Request)** –
   ```http
   GET https://api.netatmo.com/api/gethomecoachsdata
   Authorization: Bearer {{steps.Get_Auth.$return_value.access_token}}
   ```
   <img width="1195" height="645" alt="image" src="https://github.com/user-attachments/assets/95bc30b3-5bf0-4fc8-90c3-e3bca53b1268" />

4. **Return HTTP Response** – Flatten JSON into a TRMNL-friendly payload:
   ```json
   {
     "success": true,
     "temperature_c": 21.1,
     "co2_ppm": 808,
     "humidity_pct": 60,
     "noise_db": 51,
     "pressure_hpa": 993.4,
     "time_utc": 1760900442,
     "ts": "2025-10-19T18:40:42Z"
   }
   ```
   <img width="886" height="501" alt="image" src="https://github.com/user-attachments/assets/174d7a26-f2e9-47f7-8e26-7f3070221962" />


➡️ Example Trigger URL: `https://xxxxxx.m.pipedream.net`

---

## 📺 TRMNL Plugin

### Polling
- **Strategy**: Polling  
- **Polling URL**: your Pipedream trigger URL  
- **Verb**: GET  
- **Interval**: set to **hourly** to respect Netatmo’s 24/day limit.  

### Markup (Dashboard Layout)
We designed a **2×2 grid** with a footer:

```html
<div class="layout layout--col gap--none" style="height:100%; color:black;">

  <!-- Header -->
  <div class="title_bar bg--gray-75 border--h-6 rounded--none" style="color:black;">
    <span class="title" style="color:black;">Netatmo</span>
    <span class="instance" style="color:black;">
      Last updated: {{ ts | default: time_utc | date: "%Y-%m-%d %H:%M" }}
    </span>
  </div>

  <div style="display:flex; flex-direction:column; flex:1 1 auto; min-height:0;">

    <!-- 2×2 Grid -->
    <div class="grid grid--cols-2 gap--none"
         style="flex:1 1 auto; min-height:0; grid-template-rows: 1fr 1fr; grid-auto-rows: minmax(0,1fr);">

      <!-- CO₂ -->
      <div class="item {% if co2_ppm > 1100 %}bg--gray-60{% endif %} border--h-4 border--v-4"
           style="display:flex;flex-direction:column;justify-content:center;align-items:center;">
        <span class="value value--tnums value--xxxlarge" data-value-fit="true">{{ co2_ppm }}</span>
        <span class="label">ppm CO₂</span>
      </div>

      <!-- Temperature -->
      <div class="item border--h-4 border--v-4"
           style="display:flex;flex-direction:column;justify-content:center;align-items:center;">
        <span class="value value--tnums value--xxxlarge" data-value-fit="true">{{ temperature_c }}</span>
        <span class="label">°C Temperature</span>
      </div>

      <!-- Noise -->
      <div class="item border--h-4 border--v-4"
           style="display:flex;flex-direction:column;justify-content:center;align-items:center;">
        <span class="value value--tnums value--xxxlarge" data-value-fit="true">{{ noise_db }}</span>
        <span class="label">dB Noise</span>
      </div>

      <!-- Humidity -->
      <div class="item border--h-4 border--v-4"
           style="display:flex;flex-direction:column;justify-content:center;align-items:center;">
        <span class="value value--tnums value--xxxlarge" data-value-fit="true">{{ humidity_pct }}</span>
        <span class="label">% Humidity</span>
      </div>
    </div>

    <!-- Footer: Pressure -->
    <div class="item {% if pressure_hpa < 995 %}bg--gray-70{% endif %} border--h-6 rounded--none"
         style="display:flex;flex-direction:column;justify-content:center;align-items:center;">
      <span class="value value--tnums value--xlarge">{{ pressure_hpa }}</span>
      <span class="label">hPa Pressure{% if pressure_hpa < 995 %} — Low{% endif %}</span>
    </div>

  </div>
</div>
```

### Features
- **CO₂ Warning:** background shade when >1100 ppm.  
- **Pressure Low:** background shade when <995 hPa.  
- **Rounded Corners & Borders:** outer edges smooth, inner dividers crisp.  
- **Auto Time Stamp:** last update shown in header.  

---

## 📋 Summary
- Netatmo rate limit = 24/day → **hourly calls max**.  
- Pipedream handles token refresh + data fetch + JSON flattening.  
- TRMNL plugin polls the Pipedream endpoint hourly.  
- Dashboard shows CO₂, Temp, Noise, Humidity in quads, Pressure in footer, with alerts.  
<img width="939" height="183" alt="image" src="https://github.com/user-attachments/assets/313d949f-5640-4c9c-bc36-6ce8528622b6" />


---

## ✅ Recreate Checklist
1. Get Netatmo API credentials + tokens.  
2. Build Pipedream workflow (auth refresh → fetch → return JSON).  
3. Copy Trigger URL into TRMNL Plugin (polling).  
4. Paste the markup into TRMNL Plugin editor.  
5. Set polling to **hourly**.  
6. Enjoy your live Netatmo dashboard!
