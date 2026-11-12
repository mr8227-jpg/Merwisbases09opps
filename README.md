# Merwisbases09opps
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Neon Drone: Debug Protocol</title>
<style>
  html, body { margin: 0; height: 100%; background: #03050d; }
  body { display: flex; align-items: center; justify-content: center; }
  #wrap { width: min(100vw, calc(100vh * 4 / 3)); aspect-ratio: 4 / 3; }
  canvas { width: 100%; height: 100%; display: block; border-radius: 6px; box-shadow: 0 0 50px rgba(25,240,255,0.16); }
</style>
</head>
<body>
<div id="wrap"><canvas id="game" width="800" height="600"></canvas></div>
<script>
// ===== NEON DRONE: DEBUG PROTOCOL (Version 2) =====
// Static game: HTML + CSS + JavaScript only. Sound is generated in code (Web Audio), no files needed.

const W = 800, H = 600, WIN_TIME = 60;
const COL = { cyan: '#19f0ff', mag: '#ff2fa8', lime: '#9dff3a', org: '#ff9a2f', ink: '#d8f6ff', dim: '#8fb4c4' };
const FACTS = {
  virus: 'A virus attaches to a legitimate file and spreads when that file is run.',
  worm: 'A worm copies itself across networks on its own, with no host file needed.',
  trojan: 'A trojan hides inside software that looks useful, so people install it willingly.'
};
const cv = document.getElementById('game'), ctx = cv.getContext('2d');

let state = 'menu', keys = {};
let drone, bullets = [], bugs = [], parts = [], pickups = [];
let t = 0, kills = 0, bonus = 0, chain = 0, comboT = 0, spawnT = 0, fireT = 0, shieldT = 0;
let waveDone = {}, fact = FACTS.virus, finalT = 0, killer = 'virus', shake = 0, flash = 0, hi = 0, newBest = false;
let clk = 0, muted = false, AC = null;
try { hi = +localStorage.getItem('ndHi') || 0; } catch (e) {}

const rand = (a, b) => a + Math.random() * (b - a);
const clamp = (v, a, b) => Math.max(a, Math.min(b, v));
const mult = () => Math.min(5, 1 + Math.floor(chain / 3));          // combo multiplier x1..x5
const getScore = () => Math.floor(finalT || t) + bonus;               // +1 per second, plus kill bonuses
const bugCol = b => b.type === 'virus' ? COL.mag : b.type === 'worm' ? COL.lime : COL.org;

// ---- background stars (3 layers move at different speeds = parallax depth) ----
const stars = [];
for (let i = 0; i < 90; i++) { const l = i % 3; stars.push({ x: rand(0, W), y: rand(0, H), v: [18, 40, 85][l], r: [0.7, 1.1, 1.7][l], a: [0.35, 0.55, 0.85][l] }); }
const vig = ctx.createRadialGradient(W / 2, H / 2, H * 0.35, W / 2, H / 2, H * 0.85);
vig.addColorStop(0, 'rgba(0,0,0,0)'); vig.addColorStop(1, 'rgba(0,0,0,0.65)');
const sky = ctx.createLinearGradient(0, 0, 0, H);
sky.addColorStop(0, '#04061a'); sky.addColorStop(1, '#0b1236');

// ---- sound (procedural) ----
function audio() {
  if (!AC) { try { AC = new (window.AudioContext || window.webkitAudioContext)(); } catch (e) {} }
  if (AC && AC.state === 'suspended') AC.resume();
}
function tone(f, d, type, vol, slide, delay) {
  if (!AC || muted) return;
  const s = AC.currentTime + (delay || 0), o = AC.createOscillator(), g = AC.createGain();
  o.type = type; o.frequency.setValueAtTime(f, s);
  if (slide) o.frequency.exponentialRampToValueAtTime(Math.max(30, f * slide), s + d);
  g.gain.setValueAtTime(vol, s); g.gain.exponentialRampToValueAtTime(0.001, s + d);
  o.connect(g); g.connect(AC.destination); o.start(s); o.stop(s + d);
}
function noise(d, vol) {
  if (!AC || muted) return;
  const n = Math.floor(AC.sampleRate * d), buf = AC.createBuffer(1, n, AC.sampleRate), ch = buf.getChannelData(0);
  for (let i = 0; i < n; i++) ch[i] = (Math.random() * 2 - 1) * (1 - i / n);
  const s = AC.createBufferSource(), g = AC.createGain();
  s.buffer = buf; g.gain.value = vol; s.connect(g); g.connect(AC.destination); s.start();
}

// ---- input ----
addEventListener('keydown', e => {
  if (['Space', 'ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight'].includes(e.code)) e.preventDefault();
  keys[e.code] = true; audio();
  if (e.code === 'KeyM') muted = !muted;
  if (e.code === 'Enter' && state !== 'playing') start();
});
addEventListener('keyup', e => { keys[e.code] = false; });
addEventListener('blur', () => { keys = {}; });
cv.addEventListener('pointerdown', () => { audio(); if (state !== 'playing') start(); });

function start() {
  drone = { x: W / 2, y: H - 80, vx: 0, vy: 0, r: 14, tilt: 0 };
  bullets = []; bugs = []; parts = []; pickups = [];
  t = 0; kills = 0; bonus = 0; chain = 0; comboT = 0; finalT = 0; shieldT = 0; flash = 0; newBest = false;
  spawnT = 0.6; fireT = 0; waveDone = { 20: false, 40: false };
  state = 'playing';
}

function endGame(won) {
  state = won ? 'won' : 'lost';
  finalT = Math.min(t, WIN_TIME);
  fact = won ? FACTS[['virus', 'worm', 'trojan'][Math.floor(rand(0, 3))]] : FACTS[killer === 'mini' ? 'trojan' : killer];
  if (getScore() > hi) { hi = getScore(); newBest = true; try { localStorage.setItem('ndHi', hi); } catch (e) {} }
  if (won) { [523, 659, 784, 1046].forEach((f, i) => tone(f, 0.25, 'triangle', 0.08, 0, i * 0.12)); }
  else { burst(drone.x, drone.y, COL.cyan, 40); shake = 14; flash = 1; noise(0.7, 0.25); tone(170, 0.6, 'sawtooth', 0.12, 0.3); }
}

// ---- enemies ----
// Virus: straight line (linear motion). Worm: sine wave path (trigonometry).
// Trojan: splits into two mini bugs when destroyed.
function spawn(x) {
  const r = Math.random();
  const type = r < 0.45 || t < 12 && r < 0.65 ? 'virus' : r < 0.75 || t < 12 ? 'worm' : 'trojan';
  const bx = clamp(x === undefined ? rand(100, W - 100) : x, 100, W - 100);
  bugs.push({
    type, baseX: bx, x: bx, y: -25, age: 0, ph: rand(0, 6.28), freq: rand(2, 3.5),
    r: type === 'worm' ? 15 : type === 'trojan' ? 19 : 17,
    speed: (type === 'worm' ? 70 : type === 'trojan' ? 60 : 90) + t * 1.2,
    amp: type === 'worm' ? rand(50, 90) : 0
  });
}

function burst(x, y, col, n) {
  for (let i = 0; i < n; i++) {
    const a = rand(0, 6.28), s = rand(60, 260);
    parts.push({ x, y, vx: Math.cos(a) * s, vy: Math.sin(a) * s, life: rand(0.3, 0.8), col, s: rand(1.5, 3.5) });
  }
}

function killBug(b) {
  b.dead = true; kills++; chain++; comboT = 2;
  bonus += 10 * mult();
  burst(b.x, b.y, bugCol(b), 16); shake = Math.max(shake, 3);
  tone(520, 0.12, 'square', 0.04, 0.4); noise(0.1, 0.07);
  if (b.type === 'trojan') {
    for (const s of [-1, 1]) bugs.push({ type: 'mini', baseX: clamp(b.x + s * 28, 30, W - 30), x: b.x, y: b.y, age: 0, ph: 0,
      freq: rand(3, 4.5), r: 10, speed: b.speed * 1.25, amp: 35 });
  } else if (Math.random() < 0.1) pickups.push({ x: b.x, y: b.y, age: 0 });
}

// ---- update ----
function update(dt) {
  t += dt;
  const L = keys.KeyA || keys.ArrowLeft, R = keys.KeyD || keys.ArrowRight, U = keys.KeyW || keys.ArrowUp;

  // Drone physics: acceleration, drag, gravity and thrust give it real momentum.
  drone.vx += (((R ? 1 : 0) - (L ? 1 : 0)) * 2200 - drone.vx * 6) * dt;
  drone.vy = clamp(drone.vy + (U ? -1100 : 380) * dt, -320, 260);
  drone.x = clamp(drone.x + drone.vx * dt, drone.r, W - drone.r);
  drone.y += drone.vy * dt;
  if (drone.y > H - drone.r) { drone.y = H - drone.r; drone.vy = 0; }
  if (drone.y < drone.r) { drone.y = drone.r; drone.vy = 0; }
  drone.tilt += (drone.vx / 365 * 0.4 - drone.tilt) * Math.min(1, 10 * dt);

  // Engine exhaust
  for (let i = 0; i < (U ? 3 : 1); i++) parts.push({ x: drone.x + rand(-3, 3), y: drone.y + 12, vx: rand(-20, 20) - drone.vx * 0.1,
    vy: rand(80, 150) + (U ? 120 : 0), life: rand(0.15, 0.3), col: U ? COL.org : COL.cyan, s: rand(1.5, 2.5) });

  // Auto-fire with muzzle flash
  fireT -= dt;
  if (fireT <= 0) {
    bullets.push({ x: drone.x, y: drone.y - 18, vy: -680 }); fireT = 0.17;
    parts.push({ x: drone.x, y: drone.y - 20, vx: 0, vy: -40, life: 0.07, col: '#ffffff', s: 5 });
  }

  // Difficulty: the gap between spawns shrinks; waves at 20 s and 40 s
  spawnT -= dt;
  if (spawnT <= 0) { spawn(); spawnT = Math.max(0.3, 1.0 - t * 0.011); }
  [20, 40].forEach(w => { if (t >= w && !waveDone[w]) { waveDone[w] = true; for (let i = 0; i < 6; i++) spawn(100 + i * 120); } });

  comboT -= dt; if (comboT <= 0) chain = 0;
  if (shieldT > 0) shieldT -= dt;

  bullets.forEach(b => { b.y += b.vy * dt; });
  bullets = bullets.filter(b => b.y > -20);
  bugs.forEach(b => { b.age += dt; b.y += b.speed * dt; b.x = b.baseX + b.amp * Math.sin(b.freq * b.age + b.ph); });
  bugs = bugs.filter(b => b.y < H + 50);
  pickups.forEach(p => { p.age += dt; p.y += 90 * dt; p.x += Math.sin(p.age * 3) * 25 * dt; });
  pickups = pickups.filter(p => p.y < H + 30);

  // Collisions: circles touch when distance between centers < sum of radii
  for (const bug of bugs) for (const bl of bullets) {
    if (bl.dead || bug.dead) continue;
    const dx = bug.x - bl.x, dy = bug.y - bl.y;
    if (dx * dx + dy * dy < (bug.r + 4) * (bug.r + 4)) { bl.dead = true; killBug(bug); }
  }
  bullets = bullets.filter(b => !b.dead);
  bugs = bugs.filter(b => !b.dead);

  for (const bug of bugs) {
    const dx = bug.x - drone.x, dy = bug.y - drone.y, rr = bug.r + drone.r - 4;
    if (dx * dx + dy * dy < rr * rr) {
      if (shieldT > 0) { shieldT = 0; killBug(bug); shake = 8; tone(200, 0.3, 'sawtooth', 0.08, 0.5); }
      else { killer = bug.type; endGame(false); break; }
    }
  }
  bugs = bugs.filter(b => !b.dead);
  pickups = pickups.filter(p => {
    const dx = p.x - drone.x, dy = p.y - drone.y;
    if (dx * dx + dy * dy < 26 * 26) { shieldT = 6; burst(p.x, p.y, COL.lime, 14); tone(660, 0.18, 'sine', 0.08, 2); return false; }
    return true;
  });
  if (state === 'playing' && t >= WIN_TIME) endGame(true);
}

// ---- drawing ----
const glow = (c, b) => { ctx.shadowColor = c; ctx.shadowBlur = b; };
const noGlow = () => { ctx.shadowBlur = 0; };
function text(s, x, y, size, col, align, bold) {
  ctx.font = (bold ? 'bold ' : '') + size + 'px "Trebuchet MS","Segoe UI",sans-serif';
  ctx.fillStyle = col; ctx.textAlign = align || 'center'; ctx.fillText(s, x, y);
}
function sphere(x, y, r, c1, c2) {
  const g = ctx.createRadialGradient(x - r * 0.35, y - r * 0.35, r * 0.1, x, y, r);
  g.addColorStop(0, c1); g.addColorStop(1, c2); ctx.fillStyle = g;
  ctx.beginPath(); ctx.arc(x, y, r, 0, 6.28); ctx.fill();
}

function drawBackground() {
  ctx.fillStyle = sky; ctx.fillRect(0, 0, W, H);
  stars.forEach(s => { ctx.globalAlpha = s.a; ctx.fillStyle = '#bfefff'; ctx.fillRect(s.x, s.y, s.r, s.r * 2.2); });
  ctx.globalAlpha = 1; ctx.strokeStyle = 'rgba(25,240,255,0.07)'; ctx.lineWidth = 1; ctx.beginPath();
  for (let x = 0; x <= W; x += 50) { ctx.moveTo(x, 0); ctx.lineTo(x, H); }
  for (let y = (clk * 30) % 50 - 50; y <= H; y += 50) { ctx.moveTo(0, y); ctx.lineTo(W, y); }
  ctx.stroke();
}

function drawDrone() {
  ctx.save(); ctx.translate(drone.x, drone.y); ctx.rotate(drone.tilt);
  if (shieldT > 0) {
    ctx.globalAlpha = shieldT < 2 && Math.sin(clk * 20) > 0 ? 0.3 : 1;
    glow(COL.lime, 20); ctx.strokeStyle = COL.lime; ctx.lineWidth = 2; ctx.fillStyle = 'rgba(157,255,58,0.08)';
    ctx.beginPath(); ctx.arc(0, 0, 27, 0, 6.28); ctx.fill(); ctx.stroke(); ctx.globalAlpha = 1;
  }
  const g = ctx.createLinearGradient(-18, 0, 18, 0);
  g.addColorStop(0, '#0b2a3a'); g.addColorStop(0.5, '#6aeaff'); g.addColorStop(1, '#0b2a3a');
  glow(COL.cyan, 18); ctx.fillStyle = g; ctx.strokeStyle = COL.cyan; ctx.lineWidth = 2;
  ctx.beginPath(); ctx.moveTo(0, -18); ctx.lineTo(7, -4); ctx.lineTo(18, 10); ctx.lineTo(8, 8); ctx.lineTo(0, 13);
  ctx.lineTo(-8, 8); ctx.lineTo(-18, 10); ctx.lineTo(-7, -4); ctx.closePath(); ctx.fill(); ctx.stroke();
  noGlow(); sphere(0, -3, 3.5, '#ffffff', '#19f0ff');
  const on = Math.sin(clk * 7) > 0;
  sphere(-15, 9, 1.8, on ? '#ff6b6b' : '#552222', '#880000'); sphere(15, 9, 1.8, on ? '#552222' : '#6bff8a', '#008822');
  ctx.restore();
}

function drawBug(b) {
  const c = bugCol(b); glow(c, 18);
  if (b.type === 'virus') {
    ctx.strokeStyle = c; ctx.lineWidth = 2.5; ctx.beginPath();
    for (let i = 0; i < 8; i++) {
      const a = i * Math.PI / 4 + b.age * 1.4;
      ctx.moveTo(b.x + Math.cos(a) * (b.r - 4), b.y + Math.sin(a) * (b.r - 4));
      ctx.lineTo(b.x + Math.cos(a) * (b.r + 5), b.y + Math.sin(a) * (b.r + 5));
    }
    ctx.stroke(); sphere(b.x, b.y, b.r - 3, '#ffb0dc', '#7a0a4a');
    sphere(b.x, b.y, 4 + Math.sin(b.age * 6), '#ffffff', '#ff2fa8');
  } else if (b.type === 'worm') {
    for (let i = 3; i >= 0; i--) {
      const x = b.baseX + b.amp * Math.sin(b.freq * (b.age - 0.09 * i) + b.ph);
      sphere(x, b.y - i * 12, b.r - i * 2.5, i === 0 ? '#dfffa8' : '#8bd62e', '#1c4a00');
    }
    noGlow(); ctx.fillStyle = '#10200a';
    ctx.beginPath(); ctx.arc(b.x - 5, b.y + 3, 2.4, 0, 6.28); ctx.arc(b.x + 5, b.y + 3, 2.4, 0, 6.28); ctx.fill();
  } else {                                                  // trojan or mini: a rotating box
    ctx.save(); ctx.translate(b.x, b.y); ctx.rotate(b.age * 1.2);
    const g = ctx.createLinearGradient(-b.r, -b.r, b.r, b.r); g.addColorStop(0, '#ffd08a'); g.addColorStop(1, '#8a3a00');
    ctx.fillStyle = g; ctx.strokeStyle = c; ctx.lineWidth = 2;
    ctx.fillRect(-b.r, -b.r, b.r * 2, b.r * 2); ctx.strokeRect(-b.r, -b.r, b.r * 2, b.r * 2);
    ctx.fillStyle = 'rgba(255,255,255,0.35)'; ctx.fillRect(-b.r * 0.5, -b.r * 0.5, b.r, b.r); ctx.restore();
  }
  noGlow();
}

function drawHUD() {
  ctx.fillStyle = 'rgba(25,240,255,0.2)'; ctx.fillRect(0, 0, W, 5);
  glow(COL.cyan, 10); ctx.fillStyle = COL.cyan; ctx.fillRect(0, 0, W * clamp(t / WIN_TIME, 0, 1), 5); noGlow();
  text('Score ' + getScore(), 18, 34, 22, COL.ink, 'left', true);
  text('Time left ' + Math.max(0, Math.ceil(WIN_TIME - t)), W - 18, 34, 22, COL.ink, 'right', true);
  if (mult() > 1) { glow(COL.mag, 12); text('x' + mult() + ' combo', W / 2, 36, 22, COL.mag, 'center', true); noGlow(); }
  if (shieldT > 0) { ctx.fillStyle = COL.lime; ctx.fillRect(18, 46, 80 * shieldT / 6, 4); }
  text('M: sound ' + (muted ? 'off' : 'on') + (hi ? '    Best ' + hi : ''), 18, H - 14, 13, COL.dim, 'left');
}

function overlay(title, col, lines) {
  ctx.fillStyle = 'rgba(3,5,13,0.8)'; ctx.fillRect(0, 0, W, H);
  glow(col, 24); text(title, W / 2, 180, 52, col, 'center', true); noGlow();
  lines.forEach((l, i) => text(l.s, W / 2, 245 + i * 34, l.size || 20, l.col || COL.ink, 'center', l.bold));
}

function draw() {
  ctx.save();
  if (shake > 0) ctx.translate(rand(-shake, shake), rand(-shake, shake));
  drawBackground();
  glow(COL.cyan, 10); ctx.strokeStyle = '#c8fbff'; ctx.lineWidth = 3;
  bullets.forEach(b => { ctx.beginPath(); ctx.moveTo(b.x, b.y); ctx.lineTo(b.x, b.y + 14); ctx.stroke(); });
  noGlow();
  pickups.forEach(p => {
    glow(COL.lime, 16); ctx.strokeStyle = COL.lime; ctx.lineWidth = 2.5;
    ctx.beginPath(); ctx.arc(p.x, p.y, 12, 0, 6.28); ctx.moveTo(p.x - 6, p.y); ctx.lineTo(p.x + 6, p.y);
    ctx.moveTo(p.x, p.y - 6); ctx.lineTo(p.x, p.y + 6); ctx.stroke(); noGlow();
  });
  bugs.forEach(drawBug);
  if (state === 'playing' || state === 'won') drawDrone();
  ctx.globalCompositeOperation = 'lighter';
  parts.forEach(p => { ctx.globalAlpha = clamp(p.life * 2.5, 0, 1); ctx.fillStyle = p.col; ctx.beginPath(); ctx.arc(p.x, p.y, p.s, 0, 6.28); ctx.fill(); });
  ctx.globalCompositeOperation = 'source-over'; ctx.globalAlpha = 1;
  ctx.restore();

  ctx.fillStyle = vig; ctx.fillRect(0, 0, W, H);
  if (flash > 0) { ctx.fillStyle = 'rgba(255,255,255,' + flash * 0.6 + ')'; ctx.fillRect(0, 0, W, H); }
  if (state === 'playing') drawHUD();
  if (state === 'menu') overlay('Neon Drone', COL.cyan, [
    { s: 'Debug Protocol', size: 26, col: COL.mag, bold: true },
    { s: 'Survive 60 seconds. Your drone fires on its own.', size: 18 },
    { s: 'A / D move left and right.  W thrusts up.  M toggles sound.', size: 18 },
    { s: 'Pink virus drops straight. Green worm weaves. Orange trojan splits in two.', size: 15, col: COL.dim },
    { s: 'Grab the green + for a one-hit shield. Quick kills build a combo.', size: 15, col: COL.dim },
    { s: hi ? 'Best score ' + hi : 'One hit and you are deleted.', size: 16, col: COL.dim },
    { s: 'Press Enter or click to start', size: 22, col: COL.cyan, bold: true }]);
  if (state === 'won' || state === 'lost') {
    const won = state === 'won';
    overlay(won ? 'System clean' : 'Drone deleted', won ? COL.lime : COL.mag, [
      { s: 'Score ' + getScore() + (newBest ? '  (new best!)' : '    Best ' + hi), size: 30, bold: true },
      { s: 'Survived ' + finalT.toFixed(1) + ' s    Bugs destroyed ' + kills, size: 18 },
      { s: fact, size: 15, col: COL.dim },
      { s: 'Press Enter or click to play again', size: 22, col: COL.cyan, bold: true }]);
  }
}

// ---- main loop ----
let last = performance.now();
function frame(now) {
  const dt = Math.min(0.033, (now - last) / 1000); last = now; clk += dt;
  stars.forEach(s => { s.y += s.v * dt; if (s.y > H) { s.y = 0; s.x = rand(0, W); } });
  if (state === 'playing') update(dt);
  parts.forEach(p => { p.x += p.vx * dt; p.y += p.vy * dt; p.life -= dt; });
  parts = parts.filter(p => p.life > 0);
  shake = Math.max(0, shake - 40 * dt); flash = Math.max(0, flash - 2.5 * dt);
  draw();
  requestAnimationFrame(frame);
}
requestAnimationFrame(frame);
</script>
</body>
</html>
