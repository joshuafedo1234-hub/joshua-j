<!--
IBM-FE - Live Weather Dashboard
Single-file (index.html) -- front-end only
Replace YOUR_API_KEY_HERE with your OpenWeatherMap API key (https://openweathermap.org/api)
Run: open the file in a browser or use a local dev server (Live Server extension or `python -m http.server`)
Features: search by city, geolocation, C/F toggle, 5-day forecast, dynamic backgrounds, caching, error handling
-->

<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>IBM-FE - Live Weather Dashboard</title>
  <style>
    :root{
      --bg:#0b1220;
      --card:#0f1724;
      --accent:#38bdf8;
      --muted:#94a3b8;
      --glass: rgba(255,255,255,0.04);
      font-family: Inter, ui-sans-serif, system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial;
    }
    *{box-sizing:border-box}
    html,body{height:100%;margin:0;background:linear-gradient(180deg,var(--bg) 0%, #071025 100%);color:#e6eef8}

    .app{min-height:100vh;display:flex;align-items:center;justify-content:center;padding:32px}
    .container{width:100%;max-width:980px;background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));border-radius:14px;padding:20px;box-shadow:0 8px 30px rgba(2,6,23,0.7);display:grid;grid-template-columns:1fr 340px;gap:18px}

    /* Left (main) */
    .main{padding:18px;background:var(--glass);border-radius:10px;}
    .controls{display:flex;gap:8px;margin-bottom:12px;align-items:center}
    .search{flex:1;display:flex}
    input[type="search"]{flex:1;padding:10px 12px;border-radius:10px;border:1px solid rgba(255,255,255,0.06);background:transparent;color:inherit;outline:none}
    button{background:transparent;border:1px solid rgba(255,255,255,0.06);padding:8px 10px;border-radius:10px;color:inherit;cursor:pointer}
    .btn-primary{background:linear-gradient(90deg,#0369a1,#06b6d4);border:none}

    .current{display:flex;gap:18px;align-items:center}
    .meta{flex:1}
    .city{font-size:20px;font-weight:600}
    .temp{font-size:64px;font-weight:700;margin:6px 0}
    .small{color:var(--muted);font-size:13px}

    .right{padding:18px;background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));border-radius:10px}
    .card{background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));padding:12px;border-radius:10px;margin-bottom:12px}

    .forecast{display:flex;gap:8px;overflow:auto;padding-bottom:6px}
    .fc-item{min-width:72px;padding:8px;border-radius:8px;text-align:center;background:rgba(255,255,255,0.02)}

    .loading{display:inline-block;width:26px;height:26px;border-radius:50%;border:3px solid rgba(255,255,255,0.08);border-top-color:var(--accent);animation:spin 1s linear infinite}
    @keyframes spin{to{transform:rotate(360deg)}}

    /* dynamic weather background */
    .sky{position:absolute;inset:0;border-radius:14px;z-index:-1;opacity:0.18;pointer-events:none}
    .sun{position:absolute;right:18px;top:18px;width:120px;height:120px;border-radius:50%;filter:blur(14px);opacity:0.85}

    footer{color:var(--muted);font-size:13px;margin-top:8px}

    /* responsive */
    @media (max-width:900px){.container{grid-template-columns:1fr;}
      .city{font-size:18px}
      .temp{font-size:44px}
    }
  </style>
</head>
<body>
  <div class="app">
    <div class="container" role="main" aria-labelledby="appTitle">

      <div class="main">
        <div style="position:relative">
          <div class="sky" id="sky"></div>
          <div class="sun" id="sun"></div>
        </div>

        <div class="controls" aria-hidden="false">
          <div class="search">
            <input id="q" type="search" placeholder="Search city or zip (e.g. Mumbai, 110001)" aria-label="Search city" />
            <button id="searchBtn" title="Search">Search</button>
          </div>
          <button id="geoBtn" title="Use my location">My location</button>
          <button id="unitBtn" title="Toggle °C/°F">°C</button>
        </div>

        <div class="card current" id="currentCard" aria-live="polite">
          <div class="meta">
            <div class="city" id="city">—</div>
            <div class="small" id="desc">Enter a city or use my location</div>
            <div class="temp" id="temp">—</div>
            <div class="small">Feels like: <span id="feels">—</span> · Humidity: <span id="hum">—</span></div>
          </div>
          <div style="min-width:120px;text-align:center">
            <div id="icon" aria-hidden="true" style="font-size:48px">☀️</div>
            <div class="small" id="wind">Wind: —</div>
          </div>
        </div>

        <div class="card">
          <div class="small">5-day forecast</div>
          <div class="forecast" id="forecast"></div>
        </div>

        <footer>Data provided by OpenWeatherMap · Demo: IBM-FE Live Weather Dashboard</footer>
      </div>

      <aside class="right" aria-label="controls and info">
        <div class="card">
          <strong>Quick tips</strong>
          <ul class="small">
            <li>Search city names (e.g. "New Delhi") or postal codes.</li>
            <li>Toggle °C/°F with the button.</li>
            <li>Click "My location" to use browser geolocation.</li>
          </ul>
        </div>

        <div class="card">
          <strong>Last update:</strong>
          <div id="lastUpdated" class="small">—</div>
        </div>

        <div class="card">
          <strong>Implementation notes</strong>
          <div class="small">Fetch current weather + 5 day / 3 hour forecast. Cache last result in localStorage for instant load.</div>
        </div>
      </aside>

    </div>
  </div>

  <script>
    // ============ CONFIG =============
    const API_KEY = 'YOUR_API_KEY_HERE'; // <-- replace with your OpenWeatherMap API key
    const CACHE_TTL = 10 * 60 * 1000; // 10 minutes
    const BASE = 'https://api.openweathermap.org/data/2.5';

    // ============ HELPERS =============
    const el = id => document.getElementById(id);
    const formatTemp = (k, unit) => {
      if (unit === 'C') return Math.round(k - 273.15) + '°C';
      return Math.round((k - 273.15) * 9/5 + 32) + '°F';
    }
    const fmtTime = ts => new Date(ts).toLocaleString();

    // debounce util
    function debounce(fn, wait=450){
      let t;
      return (...args)=>{clearTimeout(t);t=setTimeout(()=>fn.apply(this,args),wait)}
    }

    // basic cache wrapper
    function cacheKey(q){return 'weather_cache_' + q}
    function saveCache(key, data){localStorage.setItem(key, JSON.stringify({ts:Date.now(),data}))}
    function loadCache(key){
      try{
        const raw = localStorage.getItem(key); if(!raw) return null; const parsed = JSON.parse(raw);
        if(Date.now()-parsed.ts > CACHE_TTL) { localStorage.removeItem(key); return null }
        return parsed.data;
      }catch(e){return null}
    }

    // ============ DOM nodes =============
    const qInput = el('q'), searchBtn = el('searchBtn'), geoBtn = el('geoBtn'), unitBtn = el('unitBtn');
    const cityEl = el('city'), descEl = el('desc'), tempEl = el('temp'), feelsEl = el('feels'), humEl = el('hum'), windEl = el('wind');
    const iconEl = el('icon'), forecastEl = el('forecast'), lastUpdatedEl = el('lastUpdated');
    const sky = el('sky'), sun = el('sun');

    let unit = 'C';

    // ============ UI helpers =============
    function setLoading(on){
      if(on){ searchBtn.innerHTML = '<span class="loading" aria-hidden="true"></span>' }
      else { searchBtn.textContent = 'Search' }
    }

    function showError(msg){
      descEl.textContent = msg; cityEl.textContent = '—'; tempEl.textContent = '—'; feelsEl.textContent='—'; humEl.textContent='—'; windEl.textContent='—'; forecastEl.innerHTML='';
    }

    function setWeatherUI(payload){
      const {name, sys, weather, main, wind} = payload;
      cityEl.textContent = `${name}${sys && sys.country ? ', ' + sys.country : ''}`;
      descEl.textContent = weather && weather[0] ? weather[0].description : '';
      tempEl.textContent = formatTemp(main.temp, unit);
      feelsEl.textContent = formatTemp(main.feels_like, unit);
      humEl.textContent = main.humidity + '%';
      windEl.textContent = (wind.speed ? wind.speed + ' m/s' : '—');
      iconEl.textContent = emojiForWeather(weather && weather[0] ? weather[0].main : '');
      lastUpdatedEl.textContent = fmtTime(Date.now());
      setSkyBackground(weather && weather[0] ? weather[0].main : '');
    }

    function emojiForWeather(main){
      const m = main.toLowerCase();
      if(m.includes('cloud')) return '☁️';
      if(m.includes('rain') || m.includes('drizzle')) return '🌧️';
      if(m.includes('thunder')) return '⛈️';
      if(m.includes('snow')) return '❄️';
      if(m.includes('mist') || m.includes('fog')) return '🌫️';
      return '☀️';
    }

    function setSkyBackground(main){
      const m = (main || '').toLowerCase();
      if(m.includes('rain')){
        sky.style.background = 'radial-gradient(circle at 10% 20%, rgba(30,64,175,0.35), transparent), linear-gradient(180deg, rgba(2,6,23,0.9), rgba(7,10,25,0.95))';
        sun.style.display='none';
      } else if(m.includes('cloud')){
        sky.style.background = 'linear-gradient(180deg, rgba(100,116,139,0.12), rgba(2,6,23,0.9))';
        sun.style.display='block'; sun.style.background='radial-gradient(circle,#c7e3ff66,transparent)'; sun.style.filter='blur(12px)';
      } else if(m.includes('snow')){
        sky.style.background = 'linear-gradient(180deg, rgba(240,248,255,0.08), rgba(10,15,21,0.9))'; sun.style.display='none';
      } else {
        sky.style.background = 'linear-gradient(180deg, rgba(6,95,170,0.12), rgba(2,6,23,0.9))'; sun.style.display='block'; sun.style.background='radial-gradient(circle,#ffd88044,transparent)'; sun.style.filter='blur(18px)';
      }
    }

    // ============ FETCHING =============
    async function fetchWeatherByCoords(lat, lon){
      const key = cacheKey(`lat${lat}_lon${lon}`);
      const cached = loadCache(key); if(cached){ setWeatherUI(cached.current); renderForecast(cached.forecast); return }
      try{ setLoading(true);
        const curRes = await fetch(`${BASE}/weather?lat=${lat}&lon=${lon}&appid=${API_KEY}`);
        if(!curRes.ok) throw new Error('No current weather');
        const cur = await curRes.json();
        const fcRes = await fetch(`${BASE}/forecast?lat=${lat}&lon=${lon}&appid=${API_KEY}`);
        const fc = await fcRes.json();
        saveCache(key, {current:cur, forecast:fc});
        setWeatherUI(cur); renderForecast(fc);
      }catch(err){ console.error(err); showError('Unable to retrieve weather.'); }
      finally{ setLoading(false) }
    }

    async function fetchWeatherByQuery(query){
      const key = cacheKey(query.toLowerCase());
      const cached = loadCache(key); if(cached){ setWeatherUI(cached.current); renderForecast(cached.forecast); return }
      try{ setLoading(true);
        // try city name search
        const curRes = await fetch(`${BASE}/weather?q=${encodeURIComponent(query)}&appid=${API_KEY}`);
        if(!curRes.ok) throw new Error('City not found');
        const cur = await curRes.json();
        const {coord} = cur;
        const fcRes = await fetch(`${BASE}/forecast?lat=${coord.lat}&lon=${coord.lon}&appid=${API_KEY}`);
        const fc = await fcRes.json();
        saveCache(key, {current:cur, forecast:fc});
        setWeatherUI(cur); renderForecast(fc);
      }catch(err){ console.error(err); showError('City not found or API issue.'); }
      finally{ setLoading(false) }
    }

    // ============ FORECAST RENDER =============
    function renderForecast(fc){
      if(!fc || !fc.list) { forecastEl.innerHTML = '<div class="small">No forecast</div>'; return }
      // create a compact daily summary for next 5 days (pick roughly midday entries)
      const byDay = {};
      fc.list.forEach(item=>{
        const day = new Date(item.dt * 1000).toDateString();
        if(!byDay[day]) byDay[day] = [];
        byDay[day].push(item);
      });
      const days = Object.keys(byDay).slice(0,5);
      forecastEl.innerHTML = '';
      days.forEach(day=>{
        const items = byDay[day];
        // choose item closest to 12:00
        const midday = items.reduce((a,b)=> Math.abs(new Date(a.dt*1000).getHours() - 12) < Math.abs(new Date(b.dt*1000).getHours() - 12) ? a : b);
        const d = new Date(midday.dt * 1000);
        const temp = formatTemp(midday.main.temp, unit);
        const em = emojiForWeather(midday.weather[0].main);
        const div = document.createElement('div'); div.className='fc-item';
        div.innerHTML = `<div style="font-weight:600">${d.toLocaleDateString(undefined,{weekday:'short'})}</div><div style="font-size:20px">${em}</div><div class="small">${temp}</div>`;
        forecastEl.appendChild(div);
      })
    }

    // ============ EVENT HANDLERS =============
    searchBtn.addEventListener('click', ()=>{ const q = qInput.value.trim(); if(!q) { qInput.focus(); return } ; fetchWeatherByQuery(q) });
    qInput.addEventListener('keydown', (e)=>{ if(e.key === 'Enter'){ searchBtn.click() } });

    const debouncedSearch = debounce(()=>{ const q=qInput.value.trim(); if(q) fetchWeatherByQuery(q) },700);
    qInput.addEventListener('input', debouncedSearch);

    geoBtn.addEventListener('click', ()=>{
      if(!navigator.geolocation) { showError('Geolocation not supported'); return }
      setLoading(true);
      navigator.geolocation.getCurrentPosition(pos=>{ fetchWeatherByCoords(pos.coords.latitude, pos.coords.longitude) }, err=>{ showError('Unable to fetch location'); setLoading(false) })
    });

    unitBtn.addEventListener('click', ()=>{
      unit = unit === 'C' ? 'F' : 'C'; unitBtn.textContent = unit === 'C' ? '°C' : '°F';
      // re-render last data from cache if exists
      const last = Object.keys(localStorage).reverse().find(k=>k.startsWith('weather_cache_'));
      if(last){ const data = loadCache(last); if(data){ setWeatherUI(data.current); renderForecast(data.forecast) } }
    });

    // load last saved cache on startup if any
    (function init(){
      // try last city in cache
      const keys = Object.keys(localStorage).filter(k=>k.startsWith('weather_cache_'));
      if(keys.length){ const last = keys[keys.length-1]; const data = loadCache(last); if(data){ setWeatherUI(data.current); renderForecast(data.forecast); return } }

      // otherwise show a default prompt (optionally: fetch by IP or a default city)
      cityEl.textContent = 'Welcome'; descEl.textContent = 'Search a city or use My location'; tempEl.textContent='—';
    })();

    // Accessibility: keyboard shortcuts
    window.addEventListener('keydown', (e)=>{
      if(e.key === '/' && document.activeElement !== qInput){ e.preventDefault(); qInput.focus(); }
    });

    // END
  </script>
</body>
</html>
