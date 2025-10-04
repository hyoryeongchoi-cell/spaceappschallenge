<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width,initial-scale=1" />
    <title>NASA - Will It Rain On My Parade?</title>
    <style>
      body {
        font-family: Inter, system-ui, Segoe UI, Roboto, 'Helvetica Neue', Arial;
        margin: 0;
        padding: 20px;
        background: #f6f8fb;
        color: #0b1220;
      }
      .container {
        max-width: 900px;
        margin: 18px auto;
        background: #fff;
        padding: 18px;
        border-radius: 10px;
        box-shadow: 0 6px 20px rgba(8, 12, 20, 0.06);
      }
      h1 {
        margin: 0 0 12px;
        font-size: 20px;
      }
      .row {
        display: flex;
        gap: 10px;
        flex-wrap: wrap;
        margin-bottom: 8px;
      }
      input,
      select,
      button {
        padding: 8px;
        border-radius: 8px;
        border: 1px solid #d6dce6;
      }
      button {
        background: #0b74de;
        color: #fff;
        border: none;
        cursor: pointer;
      }
      .result {
        margin-top: 14px;
      }
      pre {
        background: #fafbff;
        padding: 10px;
        border-radius: 6px;
        overflow: auto;
      }
      .meta {
        font-size: 13px;
        color: #55607a;
      }
      .spinner {
        display: inline-block;
        width: 18px;
        height: 18px;
        border: 3px solid rgba(0, 0, 0, 0.1);
        border-top-color: #0b74de;
        border-radius: 50%;
        animation: rot 1s linear infinite;
      }
      @keyframes rot {
        to {
          transform: rotate(360deg);
        }
      }
    </style>
  </head>
  <body>
    <div class="container">
      <h1>WEPI - Climate Explorer</h1> 
      <div class="meta">
        Search by city name → Uses NASA POWER daily data from 01/01/2001 ~ 12/31/2024 to calculate average values for the selected month and day.
        <br>Parameters: <a href="https://power.larc.nasa.gov/data-access-viewer/" target="_blank" rel="noopener noreferrer">https://power.larc.nasa.gov/data-access-viewer/</a><br>
        ☀️ Temperature at 2 Meters<br>
        🌧️ Precipitation Corrected<br>
        💨 Wind Speed at 2 Meters<br>
        💦 Relative Humidity at 2 Meters
      </div>

      <div style="height: 12px"></div>

      <div class="row">
        <input
          id="city"
          placeholder="Enter city name (e.g., Seoul, Paris, New York)"
          style="flex: 2"
        />
        <select id="month">
          <option value="01">January</option>
          <option value="02">February</option>
          <option value="03">March</option>
          <option value="04">April</option>
          <option value="05">May</option>
          <option value="06">June</option>
          <option value="07">July</option>
          <option value="08">August</option>
          <option value="09">September</option>
          <option value="10">October</option>
          <option value="11">November</option>
          <option value="12">December</option>
        </select>
        <select id="day">
          <script>
            for (let d = 1; d <= 31; d++) {
              const dd = d < 10 ? '0' + d : '' + d;
              document.write(`<option value="${dd}">${d}</option>`);
            }
          </script>
        </select>

        <button id="btn" onclick="run()">Search</button>
        <div id="busy" style="display:none;align-items:center">
          <span class="spinner"></span
          ><span style="margin-left:8px">Loading data...</span>
        </div>
      </div>

      <div class="row">
        <div style="flex: 2; min-width: 320px">
          <h3 style="margin: 6px 0">Results:</h3>
          <div id="summaryText">Summary will appear here after search.</div>
        </div>
      </div>

      <!-- 여기에 이미지 넣기 -->
      <div style="text-align:center; margin-top: 20px;">
        <img src="worldMap.png" alt="World Map" style="max-width: 100%; height: auto; border-radius: 8px;" />
      </div>

    </div>

    <script>
      function showBusy(on) {
        document.getElementById('busy').style.display = on ? 'flex' : 'none';
        document.getElementById('btn').disabled = on;
      }

      async function run() {
        const city = document.getElementById('city').value.trim();
        if (!city) {
          alert('Please enter a city name, e.g., Seoul');
          return;
        }
        showBusy(true);

        try {
          const nomUrl = `https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(city)}&limit=1`;
          const geoRes = await fetch(nomUrl);
          const geoJson = await geoRes.json();
          if (!geoJson || geoJson.length === 0) {
            alert('Location not found. Please try another name.');
            showBusy(false);
            return;
          }
          const lat = parseFloat(geoJson[0].lat);
          const lon = parseFloat(geoJson[0].lon);

          const start = '20010101';
          const end = '20241231';
          const params = ['T2M', 'PRECTOTCORR', 'WS2M', 'RH2M'].join(',');
          const apiUrl = `https://power.larc.nasa.gov/api/temporal/daily/point?start=${start}&end=${end}&latitude=${lat}&longitude=${lon}&parameters=${params}&format=JSON&community=AG`;
          const apiRes = await fetch(apiUrl);
          if (!apiRes.ok) throw new Error('POWER API error: ' + apiRes.status);
          const powerJson = await apiRes.json();

          const month = document.getElementById('month').value;
          const day = document.getElementById('day').value;
          const targetMD = month + day;
          const raw = powerJson.properties && powerJson.properties.parameter;
          if (!raw) throw new Error('Parameters not found in POWER results.');

          const yearsData = [];
          const pars = ['T2M', 'PRECTOTCORR', 'WS2M', 'RH2M'];
          const t2mObj = raw.T2M || {};
          for (const dateKey in t2mObj) {
            if (dateKey.length >= 8) {
              const md = dateKey.slice(4, 8);
              if (md === targetMD) {
                const rec = { date: dateKey };
                for (const p of pars) {
                  const v = raw[p] && raw[p][dateKey];
                  rec[p] = v === null || v === undefined ? null : Number(v);
                }
                yearsData.push(rec);
              }
            }
          }
          if (yearsData.length === 0) {
            alert('No historical data found for this month/day combination.');
            showBusy(false);
            return;
          }

          function mean(arr) {
            const valid = arr.filter(v => v !== null && !isNaN(v));
            return valid.reduce((a, b) => a + b, 0) / Math.max(1, valid.length);
          }

          const T2M_arr = yearsData.map(r => r.T2M);
          const PRE_arr = yearsData.map(r => r.PRECTOTCORR);
          const WS_arr = yearsData.map(r => r.WS2M);
          const RH_arr = yearsData.map(r => r.RH2M);

          const stats = {
            years: yearsData.length,
            T2M_mean: mean(T2M_arr),
            PRE_mean: mean(PRE_arr),
            WS_mean: mean(WS_arr),
            RH_mean: mean(RH_arr),
          };

          const summaryEl = document.getElementById('summaryText');
          summaryEl.innerHTML = `
            <strong>${city}</strong> <small style="font-size: 0.8em; color: #555;">(Based on the data from 2001~2024)</small><br>
            • Average Temperature: ${
              isFinite(stats.T2M_mean) ? stats.T2M_mean.toFixed(1) + ' °C' : 'No data'
            }<br>
            • Average Precipitation: ${
              isFinite(stats.PRE_mean) ? stats.PRE_mean.toFixed(2) + ' mm' : 'No data'
            }<br>
            • Average Wind Speed: ${
              isFinite(stats.WS_mean) ? stats.WS_mean.toFixed(2) + ' m/s' : 'No data'
            }<br>
            • Average Relative Humidity: ${
              isFinite(stats.RH_mean) ? stats.RH_mean.toFixed(1) + ' %' : 'No data'
            }<br>
          `;

        } catch (err) {
          console.error(err);
          alert('An error occurred: ' + err.message);
        } finally {
          showBusy(false);
        }
      }
    </script>
  </body>
</html>
