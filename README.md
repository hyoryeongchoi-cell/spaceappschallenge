<!DOCTYPE html>
<html lang="ko">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width,initial-scale=1" />
    <title>NASA Weather Likelihood — Demo</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
      body{font-family:Inter,system-ui,Segoe UI,Roboto,'Helvetica Neue',Arial;margin:0;padding:20px;background:#f6f8fb;color:#0b1220}
      .container{max-width:900px;margin:18px auto;background:#fff;padding:18px;border-radius:10px;box-shadow:0 6px 20px rgba(8,12,20,0.06)}
      h1{margin:0 0 12px;font-size:20px}
      .row{display:flex;gap:10px;flex-wrap:wrap;margin-bottom:8px}
      input,select,button{padding:8px;border-radius:8px;border:1px solid #d6dce6}
      button{background:#0b74de;color:#fff;border:none;cursor:pointer}
      .result{margin-top:14px}
      pre{background:#fafbff;padding:10px;border-radius:6px;overflow:auto}
      .meta{font-size:13px;color:#55607a}
      .download{margin-top:8px}
      canvas{max-width:100%}
      .spinner{display:inline-block;width:18px;height:18px;border:3px solid rgba(0,0,0,0.1);border-top-color:#0b74de;border-radius:50%;animation:rot 1s linear infinite}
      @keyframes rot{to{transform:rotate(360deg)}}
    </style>
  </head>
  <body>
    <div class="container">
      <h1>NASA Weather Likelihood (데모)</h1>
      <div class="meta">
        설명: 도시 이름으로 위치 찾기 → NASA POWER의 2001–2020 일별 자료를
        사용해 선택한 월·일의 과거 관측치로 확률 계산
      </div>

      <div style="height:12px"></div>

      <div class="row">
        <input
          id="city"
          placeholder="도시 이름 입력 (예: Seoul, Paris, New York)"
          style="flex:2"
        />
        <select id="month">
          <option value="01">01월</option>
          <option value="02">02월</option>
          <option value="03">03월</option>
          <option value="04">04월</option>
          <option value="05">05월</option>
          <option value="06">06월</option>
          <option value="07">07월</option>
          <option value="08">08월</option>
          <option value="09">09월</option>
          <option value="10">10월</option>
          <option value="11">11월</option>
          <option value="12">12월</option>
        </select>
        <select id="day">
          <!-- 1..31 -->
          <script>
            for(let d=1; d<=31; d++){
              const dd = d<10? '0'+d : ''+d;
              document.write(`<option value="${dd}">${d}일</option>`);
            }
          </script>
        </select>

        <button id="btn" onclick="run()">검색</button>
        <div id="busy" style="display:none;align-items:center">
          <span class="spinner"></span
          ><span style="margin-left:8px">데이터 로딩 중...</span>
        </div>
      </div>

      <div class="row">
        <div style="flex:1;min-width:260px">
          <h3 style="margin:6px 0">설정 (임계값)</h3>
          <div class="meta">
            원하면 값을 바꿔서 '매우' 판단 기준을 조정하세요.
          </div>
          <div style="margin-top:6px">
            <label>매우 덥다 (°C): </label
            ><input id="th_hot" value="30" style="width:70px" />
          </div>
          <div style="margin-top:6px">
            <label>매우 춥다 (°C): </label
            ><input id="th_cold" value="5" style="width:70px" />
          </div>
          <div style="margin-top:6px">
            <label>강수(매우 비) (mm/day): </label
            ><input id="th_rain" value="5" style="width:70px" />
          </div>
          <div style="margin-top:6px">
            <label>바람(매우 강함) (m/s): </label
            ><input id="th_wind" value="8" style="width:70px" />
          </div>
          <div style="margin-top:6px">
            <label>습도(매우 불쾌) (%): </label
            ><input id="th_hum" value="70" style="width:70px" />
          </div>
        </div>

        <div style="flex:2;min-width:320px">
          <h3 style="margin:6px 0">결과</h3>
          <div id="summaryText">검색 후 요약이 여기에 표시됩니다.</div>
          <div class="download" id="downloadArea"></div>
        </div>
      </div>

      <div class="result">
        <canvas id="chart" height="160"></canvas>
      </div>

      <div style="margin-top:12px">
        <small class="meta"
          >데이터 출처: NASA POWER (daily). 위치검색: OpenStreetMap Nominatim
          (무료 공용 서비스). (데모 목적으로 2001–2020 사용)</small
        >
      </div>
    </div>

    <script>
      /* 1) geocode with Nominatim (OpenStreetMap)
         2) call NASA POWER daily API to fetch 2001-01-01 .. 2020-12-31 for selected parameters
         3) filter by selected month/day across years -> compute stats
         NOTE: This is a demo: for heavy/production use, consider server-side proxy, caching, or POWER's climatology endpoints.
      */

      const chartCtx = document.getElementById('chart').getContext('2d');
      let myChart = null;

      function showBusy(on){
        document.getElementById('busy').style.display = on ? 'flex':'none';
        document.getElementById('btn').disabled = on;
      }

      async function run(){
        const city = document.getElementById('city').value.trim();
        if(!city){ alert('도시 이름을 입력하세요. 예: Seoul'); return; }
        showBusy(true);
        // 1. Geocode (Nominatim)
        try {
          const nomUrl = `https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(city)}&limit=1`;
          // Nominatim 사용 정책에 따라 사용자 에이전트가 필요하나, 브라우저 fetch는 기본 header로 동작.
          const geoRes = await fetch(nomUrl);
          const geoJson = await geoRes.json();
          if(!geoJson || geoJson.length===0){ alert('위치를 찾을 수 없습니다. 다른 이름으로 시도하세요.'); showBusy(false); return; }
          const lat = parseFloat(geoJson[0].lat);
          const lon = parseFloat(geoJson[0].lon);
          // 2. NASA POWER call (Daily) — 기간: 2001-01-01 ~ 2020-12-31 (데모)
          // Parameters: T2M (°C), PRECTOTCORR (mm/day), WS2M (m/s), CLOUD_AMT_DAY (percent), RH2M (relative humidity % or 0-1)
          // NOTE: 일부 파라미터 네이밍/단위는 POWER 문서에 따름.
          const start = '20010101';
          const end = '20201231';
          const params = ['T2M','PRECTOTCORR','WS2M','CLOUD_AMT_DAY','RH2M'].join(',');
          const apiUrl = `https://power.larc.nasa.gov/api/temporal/daily/point?start=${start}&end=${end}&latitude=${lat}&longitude=${lon}&parameters=${params}&format=JSON&community=AG`;
          const apiRes = await fetch(apiUrl);
          if(!apiRes.ok){ throw new Error('POWER API 에러: ' + apiRes.status); }
          const powerJson = await apiRes.json();
          // 3. parse daily series
          // POWER returns properties.parameter.<PARAM> : { '20010101': value, ... }
          const month = document.getElementById('month').value; // '01'
          const day = document.getElementById('day').value; // '01'..'31'
          const targetMD = month + day; // e.g. '0704'
          const raw = powerJson.properties && powerJson.properties.parameter;
          if(!raw){ throw new Error('POWER 결과에서 parameter를 찾을 수 없습니다.'); }
          // build array of records for each year where that month/day exists
          const yearsData = []; // array of {date, T2M, PRECTOTCORR, WS2M, CLOUD_AMT_DAY, RH2M}
          const pars = ['T2M','PRECTOTCORR','WS2M','CLOUD_AMT_DAY','RH2M'];
          // get the list of dates from one parameter (T2M)
          const t2mObj = raw.T2M || {};
          for(const dateKey in t2mObj){
            if(dateKey.length>=8){
              const md = dateKey.slice(4,8);
              if(md === targetMD){
                const rec = { date: dateKey };
                for(const p of pars){
                  const v = raw[p] && raw[p][dateKey];
                  rec[p] = (v === null || v === undefined) ? null : Number(v);
                }
                yearsData.push(rec);
              }
            }
          }
          if(yearsData.length===0){
            alert('해당 월·일에 대한 과거 데이터가 없습니다. (예: 2월 29일과 같은 경우)');
            showBusy(false);
            return;
          }
          // 4. compute stats & probabilities (thresholds from UI)
          const toNumber = (s)=> Number(s);
          const th_hot = toNumber(document.getElementById('th_hot').value);
          const th_cold = toNumber(document.getElementById('th_cold').value);
          const th_rain = toNumber(document.getElementById('th_rain').value);
          const th_wind = toNumber(document.getElementById('th_wind').value);
          const th_hum = toNumber(document.getElementById('th_hum').value);
          // compute means and probabilities
          function mean(arr){ const valid=arr.filter(v=>v!==null && !isNaN(v)); return valid.reduce((a,b)=>a+b,0)/Math.max(1,valid.length); }
          function pct(arr, cond){ const valid=arr.filter(v=>v!==null && !isNaN(v)); if(valid.length===0) return 0; const n=valid.filter(cond).length; return Math.round((n/valid.length)*1000)/10; }
          const T2M_arr = yearsData.map(r=>r.T2M);
          const PRE_arr = yearsData.map(r=>r.PRECTOTCORR);
          const WS_arr = yearsData.map(r=>r.WS2M);
          const CLD_arr = yearsData.map(r=>r.CLOUD_AMT_DAY);
          const RH_arr = yearsData.map(r=>r.RH2M);
          const stats = {
            years: yearsData.length,
            T2M_mean: mean(T2M_arr),
            PRE_mean: mean(PRE_arr),
            WS_mean: mean(WS_arr),
            CLD_mean: mean(CLD_arr),
            RH_mean: mean(RH_arr),
            prob_hot: pct(T2M_arr, v=>v>th_hot),
            prob_cold: pct(T2M_arr, v=>v<th_cold),
            prob_rain: pct(PRE_arr, v=>v>th_rain),
            prob_wind: pct(WS_arr, v=>v>th_wind),
            prob_hum: pct(RH_arr, v=>v> ( (th_hum>1 && th_hum<=100) ? th_hum : th_hum ) ) // RH likely percent
          };
          // 5. render summary text
          const summaryEl = document.getElementById('summaryText');
          const date_disp = `${Number(month)}월 ${Number(day)}일`;
          summaryEl.innerHTML = `
            <strong>${city}</strong> — ${date_disp} (과거 ${stats.years}개 연도 기준)<br>
            • 평균 기온: ${isFinite(stats.T2M_mean)? stats.T2M_mean.toFixed(1) + ' °C' : '데이터없음'}<br>
            • 평균 강수: ${isFinite(stats.PRE_mean)? stats.PRE_mean.toFixed(2) + ' mm' : '데이터없음'}<br>
            • 평균 풍속: ${isFinite(stats.WS_mean)? stats.WS_mean.toFixed(2) + ' m/s' : '데이터없음'}<br>
            • 평균 구름량: ${isFinite(stats.CLD_mean)? stats.CLD_mean.toFixed(1) + ' %' : '데이터없음'}<br>
            • 평균 상대습도: ${isFinite(stats.RH_mean)? stats.RH_mean.toFixed(1) + (stats.RH_mean<=1? ' (0-1 scale?)':' %') : '데이터없음'}<br><br>
            <strong>임계값 기반 확률</strong> (예: '매우 덥다' = T2M > ${th_hot}°C)<br>
            • 매우 덥다: ${stats.prob_hot}%<br>
            • 매우 춥다: ${stats.prob_cold}%<br>
            • 매우 비가 옴: ${stats.prob_rain}%<br>
            • 매우 바람: ${stats.prob_wind}%<br>
            • 매우 습함: ${stats.prob_hum}%<br>
          `;
          // 6. draw chart (probabilities)
          const labels = ['매우 덥다','매우 춥다','매우 비','매우 바람','매우 습함'];
          const dataVals = [stats.prob_hot, stats.prob_cold, stats.prob_rain, stats.prob_wind, stats.prob_hum];
          if(myChart) myChart.destroy();
          myChart = new Chart(chartCtx, {
            type: 'bar',
            data: {
              labels: labels,
              datasets: [{
                label: '확률 (%)',
                data: dataVals,
              }]
            },
            options: {
              scales: { y: { beginAtZero:true, max:100 } },
              plugins: { legend: { display:false } }
            }
          });
          // 7. CSV download generator
          const csvHeader = ['date','T2M_C','PRECTOTCORR_mm','WS2M_m_s','CLOUD_AMT_DAY_percent','RH2M'];
          const csvRows = [csvHeader.join(',')];
          for(const r of yearsData){
            csvRows.push([r.date, r.T2M, r.PRECTOTCORR, r.WS2M, r.CLOUD_AMT_DAY, r.RH2M].map(v=> v===null? '': v).join(','));
          }
          const csvString = csvRows.join('\n');
          const blob = new Blob([csvString], {type: 'text/csv;charset=utf-8;'});
          const url = URL.createObjectURL(blob);
          document.getElementById('downloadArea').innerHTML = `<a href="${url}" download="${city.replace(/\s+/g,'_')}_${month}${day}_history.csv">CSV 다운로드 (과거 관측값)</a>`;
        } catch(err){
          console.error(err);
          alert('오류가 발생했습니다: ' + err.message);
        } finally {
          showBusy(false);
        }
      }
    </script>
  </body>
</html>
