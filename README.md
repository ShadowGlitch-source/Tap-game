<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Tap the Target</title>
<style>
  :root {
    --bg: #0f1720;
    --card: #111827;
    --accent: #ff6b6b;
    --muted: #94a3b8;
    --glass: rgba(255,255,255,0.03);
  }
  * { box-sizing: border-box; }
  html,body { height:100%; margin:0; font-family:system-ui,-apple-system,Segoe UI,Roboto,"Helvetica Neue",Arial; background:var(--bg); color:#fff; -webkit-tap-highlight-color: transparent; }
  .wrap {
    display:flex; flex-direction:column; align-items:center; justify-content:flex-start;
    gap:12px; padding:18px; min-height:100vh;
  }

  header { width:100%; max-width:900px; text-align:center; margin-top:6px; }
  h1 { margin:0; font-size:20px; letter-spacing:0.2px; }
  p.lead { margin:6px 0 0; color:var(--muted); font-size:13px; }

  .hud {
    width:100%; max-width:900px; display:flex; gap:8px; justify-content:space-between; align-items:center; margin-top:10px;
  }
  .badge {
    background:var(--glass); padding:8px 12px; border-radius:10px; font-size:16px; min-width:90px; text-align:center;
  }
  #startBtn {
    background:linear-gradient(180deg,#22c55e,#16a34a); border:0; color:#fff; padding:10px 14px; border-radius:10px; font-size:16px;
  }

  .arena {
    position:relative; width:100%; max-width:900px; height:60vh; min-height:320px; margin-top:10px;
    background:linear-gradient(180deg, rgba(255,255,255,0.02), transparent);
    border-radius:14px; overflow:hidden; display:flex; align-items:center; justify-content:center;
  }

  #target {
    position:absolute; width:72px; height:72px; border-radius:50%;
    display:flex; align-items:center; justify-content:center; font-weight:700; user-select:none;
    box-shadow:0 6px 14px rgba(0,0,0,0.5), inset 0 -6px 18px rgba(255,255,255,0.03);
    transform:translate(-50%,-50%); touch-action:none;
  }

  .target-inner { width:100%; height:100%; border-radius:50%; display:flex; align-items:center; justify-content:center; }
  .scoreboard { width:100%; max-width:900px; display:flex; gap:10px; justify-content:space-between; align-items:center; margin-top:10px; }
  .muted { color:var(--muted); font-size:13px; }

  /* small screens */
  @media (max-width:420px) {
    #target { width:64px; height:64px; }
    .badge { font-size:14px; padding:8px 10px; min-width:76px; }
  }
</style>
</head>
<body>
  <div class="wrap">
    <header>
      <h1>Tap the Target!</h1>
      <p class="lead">Tap the circle as many times as you can in 30 seconds. Try to beat your high score!</p>
    </header>

    <div class="hud" aria-hidden="false">
      <div class="badge" id="score">Score: 0</div>
      <div style="display:flex;gap:8px;">
        <div class="badge" id="timer">Time: 30s</div>
        <div class="badge muted" id="high">High: 0</div>
      </div>
      <button id="startBtn" aria-live="polite">Start</button>
    </div>

    <div class="arena" id="arena" role="application" aria-label="Game arena">
      <!-- target appears here -->
      <div id="target" style="display:none;">
        <div class="target-inner" id="targetInner"></div>
      </div>
    </div>

    <div class="scoreboard">
      <div class="muted">Tip: Tap quickly and move your thumb — target moves each tap.</div>
      <div class="muted">Made with HTML/CSS/JS</div>
    </div>
  </div>

<script>
(() => {
  const arena = document.getElementById('arena');
  const target = document.getElementById('target');
  const targetInner = document.getElementById('targetInner');
  const scoreEl = document.getElementById('score');
  const timerEl = document.getElementById('timer');
  const highEl = document.getElementById('high');
  const startBtn = document.getElementById('startBtn');

  let score = 0;
  let timeLeft = 30;
  let running = false;
  let timerId = null;

  // Config
  const baseSize = 72;          // px on larger devices; CSS scales responsively
  const initialAppearDelay = 300; // ms before first move

  // Colors (random-ish)
  const colors = ['#ff6b6b','#f97316','#f59e0b','#34d399','#60a5fa','#7c3aed','#fb7185','#06b6d4'];

  // Load highscore
  const HS_KEY = 'tap_target_highscore_v1';
  function loadHigh() {
    const val = localStorage.getItem(HS_KEY);
    const high = val ? Number(val) : 0;
    highEl.textContent = 'High: ' + high;
    return high;
  }
  function saveHigh(v) { localStorage.setItem(HS_KEY, String(v)); }

  loadHigh();

  function randColor() {
    return colors[Math.floor(Math.random()*colors.length)];
  }

  function randomPosition(size) {
    // compute available area inside arena
    const rect = arena.getBoundingClientRect();
    const maxX = Math.max(8, rect.width - size - 8);
    const maxY = Math.max(8, rect.height - size - 8);
    const x = Math.random() * maxX + rect.left + size/2 + 4;
    const y = Math.random() * maxY + rect.top + size/2 + 4;
    return { x, y };
  }

  function setTargetSize(px) {
    target.style.width = px + 'px';
    target.style.height = px + 'px';
  }

  function placeTarget() {
    // set dynamic size a bit based on viewport
    const vw = Math.max(document.documentElement.clientWidth || 320, 320);
    const size = Math.round(Math.max(56, Math.min(96, vw * 0.18)));
    setTargetSize(size);
    const pos = randomPosition(size);
    // pos are absolute coords relative to viewport; convert to arena container coords
    const arenaRect = arena.getBoundingClientRect();
    const localX = pos.x - arenaRect.left;
    const localY = pos.y - arenaRect.top;
    target.style.left = localX + 'px';
    target.style.top = localY + 'px';
    // style color and inner label
    targetInner.style.background = randColor();
    targetInner.textContent = ''; // can show points or icons
  }

  function showTarget() {
    target.style.display = 'block';
    // small pop animation
    target.style.transition = 'transform 140ms cubic-bezier(.2,.9,.2,1), opacity 120ms';
    target.style.opacity = '1';
    target.style.transform = 'translate(-50%,-50%) scale(1)';
  }

  function hideTarget() {
    target.style.display = 'none';
  }

  function startGame() {
    if (running) return;
    running = true;
    score = 0;
    timeLeft = 30;
    scoreEl.textContent = 'Score: ' + score;
    timerEl.textContent = 'Time: ' + timeLeft + 's';
    startBtn.disabled = true;
    startBtn.textContent = 'Playing...';
    // first show target slightly after to avoid accidental taps when starting
    setTimeout(() => {
      placeTarget();
      showTarget();
    }, initialAppearDelay);

    timerId = setInterval(() => {
      timeLeft--;
      timerEl.textContent = 'Time: ' + timeLeft + 's';
      if (timeLeft <= 0) {
        endGame();
      }
    }, 1000);
  }

  function endGame() {
    running = false;
    clearInterval(timerId);
    timerId = null;
    hideTarget();
    startBtn.disabled = false;
    startBtn.textContent = 'Start';
    // update high score
    const prevHigh = loadHigh();
    if (score > prevHigh) {
      saveHigh(score);
      highEl.textContent = 'High: ' + score;
      // small congrat
      setTimeout(()=> alert('Game over! New high score: ' + score), 80);
    } else {
      setTimeout(()=> alert('Game over! Your score: ' + score), 80);
    }
  }

  // Tapping the target counts as a hit
  function handleHit(e) {
    // prevent when not running
    if (!running) return;
    // increase score
    score++;
    scoreEl.textContent = 'Score: ' + score;
    // move target instantly to new random place
    placeTarget();
    // small vibration on supported devices
    if (navigator.vibrate) navigator.vibrate(20);
  }

  // Allow tapping anywhere to move target? No — only target taps count.
  // But add a tiny safety for touchmove to avoid accidental multiple hits.
  let lastTap = 0;
  target.addEventListener('pointerdown', (ev) => {
    const now = Date.now();
    // basic debounce to prevent double-fires
    if (now - lastTap < 50) return;
    lastTap = now;
    handleHit(ev);
  });

  // Start button
  startBtn.addEventListener('click', () => startGame());

  // Also allow keyboard "Enter" to start (useful if opened on desktop)
  document.addEventListener('keydown', (e) => {
    if (e.key === 'Enter' && !running) startGame();
  });

  // When page becomes hidden, pause the game to be fair
  document.addEventListener('visibilitychange', () => {
    if (document.hidden && running) {
      // stop early
      endGame();
      alert('Game paused because the tab became hidden. Your score was saved if it was a high score.');
    }
  });

  // On resize, keep target in view next time we place it
  window.addEventListener('resize', () => {
    if (target.style.display !== 'none') placeTarget();
  });

  // initial high display
  loadHigh();
})();
</script>
</body>
</html>