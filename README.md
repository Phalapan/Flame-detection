<!doctype html>
<html lang="th">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>🔥 Fire Detector - Sensitive Flicker Mode</title>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<style>
  body { font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Noto Sans", Arial;
         background:#0f1720; color:#e6eef8; margin:12px;}
  .card { background:#0b1220; padding:12px; border-radius:10px; 
          box-shadow:0 6px 18px rgba(0,0,0,0.6); max-width:850px; margin:auto;}
  video, canvas { border-radius:8px; max-width:100%; width:480px; height:auto; background:#111; }
  .controls { display:flex; gap:8px; flex-wrap:wrap; margin-top:10px;}
  button { background:#2563eb; color:white; border:none; padding:8px 12px; border-radius:8px; cursor:pointer;}
  button.stop { background:#ef4444; }
  label { font-size:13px; }
  .status { padding:8px; border-radius:8px; margin-top:10px; font-weight:600; }
  .status.safe { background:#052e14; color:#86efac;}
  .status.fire { background:#2b0400; color:#ffb3a1;}
  .row { display:flex; align-items:center; gap:8px; flex-wrap:wrap;}
  input[type=range]{ width:180px;}
  canvas#chart { width:100%; height:200px; margin-top:15px;}
  .small { font-size:13px; color:#9fb0cc; }
</style>
</head>
<body>
<div class="card">
  <h2>🔥 Fire Detector — Sensitive Flicker Mode</h2>

  <video id="video" autoplay muted playsinline></video>
  <canvas id="canvas" width="480" height="360" style="display:none"></canvas>

  <div class="controls">
    <button id="startBtn">เริ่มกล้อง & ตรวจจับ</button>
    <button id="stopBtn" class="stop" disabled>หยุด</button>
    <button id="testAlarm">ทดสอบเสียงเตือน</button>
  </div>

  <div style="margin-top:8px;">
    <div class="row">
      <label>Threshold สี (0–255): <span id="colorThreshVal">150</span></label>
      <input id="colorThresh" type="range" min="80" max="255" value="150">
    </div>
    <div class="row">
      <label>พิกเซลไฟ ≥ (%) : <span id="pixelRatioVal">5</span></label>
      <input id="pixelRatio" type="range" min="1" max="25" value="5">
    </div>
    <div class="row">
      <label>Flicker ≥ : <span id="flickerVal">200</span></label>
      <input id="flicker" type="range" min="20" max="2000" value="200">
    </div>
  </div>

  <div id="statusBox" class="status safe">สถานะ: ยังไม่ได้เริ่ม</div>

  <canvas id="chart"></canvas>

  <div class="small">*โหมดนี้ตรวจจับไวขึ้น — แสงกระพริบเพียงเล็กน้อยก็อาจนับเป็นเปลวไฟ</div>
</div>

<script>
(async function(){
  const video = document.getElementById('video');
  const canvas = document.getElementById('canvas');
  const ctx = canvas.getContext('2d');
  const startBtn = document.getElementById('startBtn');
  const stopBtn = document.getElementById('stopBtn');
  const testAlarm = document.getElementById('testAlarm');
  const statusBox = document.getElementById('statusBox');

  const colorThresh = document.getElementById('colorThresh');
  const pixelRatio = document.getElementById('pixelRatio');
  const flicker = document.getElementById('flicker');
  const colorThreshVal = document.getElementById('colorThreshVal');
  const pixelRatioVal = document.getElementById('pixelRatioVal');
  const flickerVal = document.getElementById('flickerVal');

  colorThresh.oninput = () => colorThreshVal.textContent = colorThresh.value;
  pixelRatio.oninput = () => pixelRatioVal.textContent = pixelRatio.value;
  flicker.oninput = () => flickerVal.textContent = flicker.value;

  // Chart.js setup
  const chartCtx = document.getElementById('chart').getContext('2d');
  const chartData = {
    labels: [],
    datasets: [
      { label: '% พิกเซลไฟ', data: [], borderColor: '#f97316', backgroundColor: 'rgba(249,115,22,0.2)', tension: 0.3 },
      { label: 'Flicker Δ', data: [], borderColor: '#22d3ee', backgroundColor: 'rgba(34,211,238,0.2)', tension: 0.3 }
    ]
  };
  const chart = new Chart(chartCtx, {
    type: 'line',
    data: chartData,
    options: {
      responsive: true,
      animation: false,
      scales: { y: { beginAtZero: true, grid: { color: '#1e293b' }, ticks: { color: '#cbd5e1' } } },
      plugins: { legend: { labels: { color: '#e6eef8' } } }
    }
  });

  // Alarm sound
  const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  function playAlarm(duration = 1000) {
    const o = audioCtx.createOscillator();
    const g = audioCtx.createGain();
    o.type = 'square';
    o.frequency.value = 700;
    g.gain.value = 0.001;
    o.connect(g);
    g.connect(audioCtx.destination);
    o.start();
    g.gain.exponentialRampToValueAtTime(0.3, audioCtx.currentTime + 0.05);
    o.frequency.linearRampToValueAtTime(300, audioCtx.currentTime + duration / 1000);
    setTimeout(() => {
      g.gain.exponentialRampToValueAtTime(0.0001, audioCtx.currentTime + 0.05);
      setTimeout(() => o.stop(), 100);
    }, duration);
  }
  testAlarm.onclick = () => playAlarm(800);

  let stream = null, rafId = null, running = false;
  let lastBrightness = 0;

  function markStatus(isFire, reason = '') {
    if (isFire) {
      statusBox.className = 'status fire';
      statusBox.textContent = '🔥 พบไฟ — ' + reason;
    } else {
      statusBox.className = 'status safe';
      statusBox.textContent = '🟢 ไม่พบไฟ — ' + reason;
    }
  }

  function stopProcess() {
    running = false;
    if (rafId) cancelAnimationFrame(rafId);
    if (stream) stream.getTracks().forEach(t => t.stop());
    startBtn.disabled = false;
    stopBtn.disabled = true;
    markStatus(false, 'หยุดตรวจจับแล้ว');
  }
  stopBtn.onclick = stopProcess;

  startBtn.onclick = async () => {
    await audioCtx.resume();
    try {
      stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' }, audio: false });
    } catch (e) {
      alert('ไม่สามารถเปิดกล้อง: ' + e.message);
      return;
    }
    video.srcObject = stream;
    await video.play();

    startBtn.disabled = true;
    stopBtn.disabled = false;
    running = true;
    canvas.width = 480;
    canvas.height = 360;
    chartData.labels.length = 0;
    chartData.datasets[0].data.length = 0;
    chartData.datasets[1].data.length = 0;
    lastBrightness = 0;
    process();
  };

  function process() {
    if (!running) return;
    ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
    const img = ctx.getImageData(0, 0, canvas.width, canvas.height);
    const data = img.data;
    let firePixels = 0, brightSum = 0;
    for (let i = 0; i < data.length; i += 16) {
      const r = data[i], g = data[i + 1], b = data[i + 2];
      const bright = 0.299 * r + 0.587 * g + 0.114 * b;
      brightSum += bright;
      const rHigh = r >= parseInt(colorThresh.value);
      const rDom = r > g * 1.1 && r > b * 1.1;
      const orange = r > 150 && g > 80 && b < 120 && (r - b) > 50;
      if ((rHigh && rDom) || orange) firePixels++;
    }
    const avgBright = brightSum / ((data.length / 4) / 4);
    const delta = Math.abs(avgBright - lastBrightness);
    lastBrightness = avgBright;

    const percent = (firePixels / ((data.length / 4) / 4)) * 100;
    const flickerNow = delta;

    chartData.labels.push('');
    chartData.datasets[0].data.push(percent);
    chartData.datasets[1].data.push(flickerNow);
    if (chartData.labels.length > 50) {
      chartData.labels.shift();
      chartData.datasets[0].data.shift();
      chartData.datasets[1].data.shift();
    }
    chart.update();

    const pth = parseFloat(pixelRatio.value);
    const fth = parseFloat(flicker.value);
    const isFire = (percent >= pth && flickerNow >= fth);
    const reason = `พิกเซลไฟ ${percent.toFixed(1)}% | flickerΔ ${flickerNow.toFixed(1)}`;
    markStatus(isFire, reason);
    if (isFire) playAlarm(800);
    rafId = requestAnimationFrame(process);
  }

  window.addEventListener('beforeunload', stopProcess);
})();
</script>
</body>
</html>

