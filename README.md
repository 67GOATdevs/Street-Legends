const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const scoreEl = document.getElementById('score');
const bestEl = document.getElementById('best');
const speedEl = document.getElementById('speed');
const overlay = document.getElementById('overlay');
const startButton = document.getElementById('startButton');

const STATE = {
  running: false,
  ended: false,
  time: 0,
  lastStamp: 0,
  score: 0,
  best: Number(localStorage.getItem('streetLegendsBest') || 0),
  roadScroll: 0,
  obstacleTimer: 0,
};

const keys = {
  left: false,
  right: false,
  accelerate: false,
  brake: false,
};

const road = {
  x: canvas.width / 2,
  width: 420,
  left: canvas.width / 2 - 210,
  right: canvas.width / 2 + 210,
};

const player = {
  width: 48,
  height: 76,
  x: canvas.width / 2,
  y: canvas.height - 110,
  speed: 240,
  maxSpeed: 520,
  minSpeed: 160,
  turnSpeed: 350,
};

const obstacles = [];

function resetGame() {
  STATE.running = true;
  STATE.ended = false;
  STATE.score = 0;
  STATE.time = 0;
  STATE.roadScroll = 0;
  STATE.obstacleTimer = 0;
  player.x = canvas.width / 2;
  player.speed = 240;
  obstacles.length = 0;
  overlay.classList.remove('visible');
  scoreEl.textContent = '0';
  speedEl.textContent = Math.round(player.speed);
  bestEl.textContent = STATE.best;
}

function endGame() {
  STATE.running = false;
  STATE.ended = true;
  STATE.best = Math.max(STATE.best, STATE.score);
  localStorage.setItem('streetLegendsBest', String(STATE.best));
  bestEl.textContent = STATE.best;
  overlay.classList.add('visible');
  const panel = overlay.querySelector('.panel');
  panel.innerHTML = `
    <p class="eyebrow">RUN ENDED</p>
    <h1>Game Over</h1>
    <p>You scored <strong>${STATE.score}</strong> points.</p>
    <div class="controls">
      <span>Best run: ${STATE.best}</span>
    </div>
    <button id="restartButton" type="button">Ride Again</button>
  `;
  document.getElementById('restartButton').addEventListener('click', () => {
    const panelToReset = overlay.querySelector('.panel');
    panelToReset.innerHTML = `
      <p class="eyebrow">NEON RUSH</p>
      <h1>Street Legends</h1>
      <p>Thread the city traffic and survive the night ride.</p>
      <div class="controls">
        <span>Steer: A / D or ← / →</span>
        <span>Boost: W / ↑</span>
        <span>Brake: S / ↓</span>
      </div>
      <button id="startButton" type="button">Start Ride</button>
    `;
    document.getElementById('startButton').addEventListener('click', resetGame);
    overlay.classList.remove('visible');
  });
}

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

function spawnObstacle() {
  const laneCount = 4;
  const width = 48;
  const height = 76;
  const gap = (road.width - width * laneCount) / (laneCount + 1);
  const lane = Math.floor(Math.random() * laneCount);
  const x = road.left + gap + (width + gap) * lane + width / 2;
  const obstacle = {
    x,
    y: -height,
    width,
    height,
    color: Math.random() > 0.5 ? '#ef4444' : '#f59e0b',
    speed: player.speed * 0.75 + 90 + Math.random() * 120,
  };

  obstacles.push(obstacle);
}

function update(dt) {
  if (!STATE.running) return;

  STATE.time += dt;
  STATE.roadScroll += player.speed * dt;
  STATE.obstacleTimer += dt;

  if (keys.left) {
    player.x -= player.turnSpeed * dt;
  }

  if (keys.right) {
    player.x += player.turnSpeed * dt;
  }

  if (keys.accelerate) {
    player.speed += 210 * dt;
  }

  if (keys.brake) {
    player.speed -= 260 * dt;
  }

  player.speed *= 0.992;
  player.speed = clamp(player.speed, player.minSpeed, player.maxSpeed);
  player.x = clamp(player.x, road.left + player.width / 2, road.right - player.width / 2);

  if (STATE.obstacleTimer > Math.max(0.7, 1.4 - player.speed / 700)) {
    STATE.obstacleTimer = 0;
    spawnObstacle();
  }

  for (let i = obstacles.length - 1; i >= 0; i -= 1) {
    const obstacle = obstacles[i];
    obstacle.y += (player.speed * 0.83 + obstacle.speed) * dt;

    if (obstacle.y > canvas.height + obstacle.height) {
      obstacles.splice(i, 1);
      STATE.score += 1;
      continue;
    }

    if (
      Math.abs(obstacle.x - player.x) < (obstacle.width + player.width) * 0.45 &&
      Math.abs(obstacle.y - player.y) < (obstacle.height + player.height) * 0.45
    ) {
      endGame();
      return;
    }
  }

  scoreEl.textContent = STATE.score;
  speedEl.textContent = Math.round(player.speed);
}

function drawBackground() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  ctx.fillStyle = '#0b1020';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  const roadOffset = STATE.roadScroll % 80;
  const roadLeft = road.x - road.width / 2;
  const roadRight = road.x + road.width / 2;

  ctx.fillStyle = '#1e293b';
  ctx.fillRect(roadLeft, 0, road.width, canvas.height);

  ctx.fillStyle = '#f8fafc';
  for (let y = -80; y < canvas.height + 80; y += 80) {
    const lineY = y + roadOffset;
    ctx.fillRect(roadLeft + 10, lineY, 12, 40);
    ctx.fillRect(roadRight - 22, lineY, 12, 40);
  }

  for (let i = 0; i < 5; i += 1) {
    const laneX = roadLeft + ((road.width / 5) * i);
    const dashWidth = 10;
    const dashHeight = 40;
    for (let y = -40; y < canvas.height + 40; y += 80) {
      const laneY = y + roadOffset;
      ctx.fillStyle = 'rgba(255,255,255,0.6)';
      ctx.fillRect(laneX + 20, laneY, dashWidth, dashHeight);
    }
  }

  ctx.fillStyle = '#0f172a';
  ctx.fillRect(roadLeft - 15, 0, 15, canvas.height);
  ctx.fillRect(roadRight, 0, 15, canvas.height);
}

function drawPlayer() {
  const carX = player.x;
  const carY = player.y;
  const carW = player.width;
  const carH = player.height;

  ctx.save();
  ctx.translate(carX, carY);

  ctx.fillStyle = '#38bdf8';
  ctx.fillRect(-carW / 2, -carH / 2, carW, carH);

  ctx.fillStyle = '#2563eb';
  ctx.fillRect(-carW * 0.18, -carH * 0.55, carW * 0.36, carH * 0.8);

  ctx.fillStyle = '#e2e8f0';
  ctx.fillRect(-carW * 0.2, -carH / 2 - 8, carW * 0.4, 12);
  ctx.fillRect(-carW * 0.2, carH / 2 - 4, carW * 0.4, 12);

  ctx.fillStyle = '#111827';
  ctx.fillRect(-carW / 2 - 4, -carH / 2 + 8, 8, 18);
  ctx.fillRect(carW / 2 - 4, -carH / 2 + 8, 8, 18);
  ctx.fillRect(-carW / 2 - 4, carH / 2 - 26, 8, 18);
  ctx.fillRect(carW / 2 - 4, carH / 2 - 26, 8, 18);

  ctx.restore();
}

function drawObstacles() {
  for (const obstacle of obstacles) {
    ctx.save();
    ctx.translate(obstacle.x, obstacle.y);
    ctx.fillStyle = obstacle.color;
    ctx.fillRect(-obstacle.width / 2, -obstacle.height / 2, obstacle.width, obstacle.height);
    ctx.fillStyle = 'rgba(15, 23, 42, 0.8)';
    ctx.fillRect(-obstacle.width * 0.2, -obstacle.height * 0.15, obstacle.width * 0.4, obstacle.height * 0.3);
    ctx.restore();
  }
}

function loop(timestamp) {
  const dt = Math.min((timestamp - STATE.lastStamp) / 1000 || 0.016, 0.033);
  STATE.lastStamp = timestamp;

  update(dt);
  drawBackground();
  drawObstacles();
  drawPlayer();

  requestAnimationFrame(loop);
}

window.addEventListener('keydown', (event) => {
  const key = event.key.toLowerCase();
  if (['arrowleft', 'a'].includes(key)) keys.left = true;
  if (['arrowright', 'd'].includes(key)) keys.right = true;
  if (['arrowup', 'w'].includes(key)) keys.accelerate = true;
  if (['arrowdown', 's'].includes(key)) keys.brake = true;

  if (event.key === ' ') {
    if (!STATE.running) resetGame();
  }
});

window.addEventListener('keyup', (event) => {
  const key = event.key.toLowerCase();
  if (['arrowleft', 'a'].includes(key)) keys.left = false;
  if (['arrowright', 'd'].includes(key)) keys.right = false;
  if (['arrowup', 'w'].includes(key)) keys.accelerate = false;
  if (['arrowdown', 's'].includes(key)) keys.brake = false;
});

startButton.addEventListener('click', resetGame);

bestEl.textContent = STATE.best;
scoreEl.textContent = '0';
speedEl.textContent = Math.round(player.speed);
requestAnimationFrame(loop);

