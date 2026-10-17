# Planeconst canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const W = canvas.width;
const H = canvas.height;

// --- Самолёт игрока ---
const plane = {
  x: W / 2 - 20,
  y: H - 100,
  w: 40,
  h: 40,
  speed: 6,
  cooldown: 0,      // задержка между выстрелами
  fireRate: 12      // кадров между выстрелами
};

// --- Управление ---
const keys = {};
document.addEventListener('keydown', e => {
  keys[e.key] = true;
  if (e.key === ' ') e.preventDefault();
});
document.addEventListener('keyup', e => keys[e.key] = false);

canvas.addEventListener('mousemove', e => {
  const rect = canvas.getBoundingClientRect();
  plane.x = e.clientX - rect.left - plane.w / 2;
  plane.y = e.clientY - rect.top - plane.h / 2;
});
canvas.addEventListener('touchmove', e => {
  e.preventDefault();
  const rect = canvas.getBoundingClientRect();
  const t = e.touches[0];
  plane.x = t.clientX - rect.left - plane.w / 2;
  plane.y = t.clientY - rect.top - plane.h / 2;
}, { passive: false });

// --- Пули игрока ---
let bullets = [];
const bulletSpeed = 10;

// --- Враги ---
let enemies = [];
let enemyTimer = 0;
let enemyInterval = 80;

// --- Взрывы (частицы) ---
let particles = [];

// --- Очки и состояние ---
let score = 0;
let gameOver = false;
const scoreEl = document.getElementById('score');
const restartBtn = document.getElementById('restart');

// --- Стрельба ---
function shoot() {
  if (plane.cooldown > 0) return;
  bullets.push({ x: plane.x + plane.w / 2 - 2, y: plane.y, w: 4, h: 12 });
  plane.cooldown = plane.fireRate;
}

// --- Спавн врага ---
function spawnEnemy() {
  const size = 30 + Math.random() * 20;
  enemies.push({
    x: Math.random() * (W - size),
    y: -size,
    size: size,
    speed: 2 + Math.random() * 2,
    hp: 1
  });
}

// --- Взрыв ---
function explode(x, y, color = '#ff9800') {
  for (let i = 0; i < 12; i++) {
    const angle = (Math.PI * 2 * i) / 12;
    particles.push({
      x, y,
      vx: Math.cos(angle) * (2 + Math.random() * 3),
      vy: Math.sin(angle) * (2 + Math.random() * 3),
      life: 30,
      color
    });
  }
}

// --- Обновление ---
function update() {
  if (gameOver) {
    updateParticles();
    return;
  }

  // движение игрока
  if (keys['ArrowLeft'] || keys['a'] || keys['A']) plane.x -= plane.speed;
  if (keys['ArrowRight'] || keys['d'] || keys['D']) plane.x += plane.speed;
  if (keys['ArrowUp'] || keys['w'] || keys['W']) plane.y -= plane.speed;
  if (keys['ArrowDown'] || keys['s'] || keys['S']) plane.y += plane.speed;
  if (keys[' ']) shoot();

  plane.x = Math.max(0, Math.min(W - plane.w, plane.x));
  plane.y = Math.max(0, Math.min(H - plane.h, plane.y));

  if (plane.cooldown > 0) plane.cooldown--;

  // пули
  bullets.forEach(b => b.y -= bulletSpeed);
  bullets = bullets.filter(b => b.y + b.h > 0);

  // спавн врагов
  enemyTimer++;
  if (enemyTimer >= enemyInterval) {
    spawnEnemy();
    enemyTimer = 0;
    enemyInterval = Math.max(30, enemyInterval - 0.5);
  }

  // движение врагов
  enemies.forEach(e => e.y += e.speed);
  enemies = enemies.filter(e => e.y < H + 50);

  // столкновение пуль с врагами
  for (let i = bullets.length - 1; i >= 0; i--) {
    const b = bullets[i];
    for (let j = enemies.length - 1; j >= 0; j--) {
      const e = enemies[j];
      if (b.x < e.x + e.size && b.x + b.w > e.x &&
          b.y < e.y + e.size && b.y + b.h > e.y) {
        bullets.splice(i, 1);
        e.hp--;
        if (e.hp <= 0) {
          explode(e.x + e.size / 2, e.y + e.size / 2);
          enemies.splice(j, 1);
          score += 10;
          scoreEl.textContent = score;
        }
        break;
      }
    }
  }

  // столкновение игрока с врагами
  for (const e of enemies) {
    if (plane.x < e.x + e.size && plane.x + plane.w > e.x &&
        plane.y < e.y + e.size && plane.y + plane.h > e.y) {
      explode(plane.x + plane.w / 2, plane.y + plane.h / 2, '#f44336');
      gameOver = true;
      restartBtn.style.display = 'block';
      break;
    }
  }

  // пассивный счёт за выживание
  score += 0.2;
  scoreEl.textContent = Math.floor(score);

  updateParticles();
}

function updateParticles() {
  particles.forEach(p => {
    p.x += p.vx;
    p.y += p.vy;
    p.life--;
  });
  particles = particles.filter(p => p.life > 0);
}

// --- Отрисовка ---
function drawPlane(x, y, w, h) {
  ctx.fillStyle = '#d32f2f';
  ctx.beginPath();
  ctx.moveTo(x + w / 2, y);
  ctx.lineTo(x + w, y + h);
  ctx.lineTo(x + w * 0.7, y + h * 0.7);
  ctx.lineTo(x + w * 0.3, y + h * 0.7);
  ctx.lineTo(x, y + h);
  ctx.closePath();
  ctx.fill();

  ctx.fillStyle = '#b71c1c';
  ctx.fillRect(x, y + h * 0.45, w, h * 0.15);

  ctx.fillStyle = '#90caf9';
  ctx.beginPath();
  ctx.arc(x + w / 2, y + h * 0.35, w * 0.12, 0, Math.PI * 2);
  ctx.fill();
}

function drawEnemy(e) {
  // простой "вражеский" самолёт — треугольник вниз
  const { x, y, size } = e;
  ctx.fillStyle = '#37474f';
  ctx.beginPath();
  ctx.moveTo(x + size / 2, y + size);     // нос вниз
  ctx.lineTo(x + size, y);                // правый верх
  ctx.lineTo(x + size * 0.7, y + size * 0.3);
  ctx.lineTo(x + size * 0.3, y + size * 0.3);
  ctx.lineTo(x, y);                       // левый верх
  ctx.closePath();
  ctx.fill();

  ctx.fillStyle = '#ff5252';
  ctx.beginPath();
  ctx.arc(x + size / 2, y + size * 0.5, size * 0.1, 0, Math.PI * 2);
  ctx.fill();
}

function draw() {
  ctx.clearRect(0, 0, W, H);

  // облака
  ctx.fillStyle = 'rgba(255,255,255,0.6)';
  for (let i = 0; i < 5; i++) {
    const t = (Date.now() / 40 + i * 130) % (H + 100);
    ctx.beginPath();
    ctx.arc(60 + i * 90, t - 50, 25, 0, Math.PI * 2);
    ctx.arc(85 + i * 90, t - 50, 30, 0, Math.PI * 2);
    ctx.arc(110 + i * 90, t - 50, 25, 0, Math.PI * 2);
    ctx.fill();
  }

  // пули
  bullets.forEach(b => {
    ctx.fillStyle = '#ffeb3b';
    ctx.fillRect(b.x, b.y, b.w, b.h);
    ctx.fillStyle = '#ff9800';
    ctx.fillRect(b.x + 1, b.y + 2, b.w - 2, b.h - 4);
  });

  // враги
  enemies.forEach(drawEnemy);

  // игрок
  drawPlane(plane.x, plane.y, plane.w, plane.h);

  // частицы
  particles.forEach(p => {
    ctx.globalAlpha = p.life / 30;
    ctx.fillStyle = p.color;
    ctx.beginPath();
    ctx.arc(p.x, p.y, 3, 0, Math.PI * 2);
    ctx.fill();
  });
  ctx.globalAlpha = 1;

  // Game Over
  if (gameOver) {
    ctx.fillStyle = 'rgba(0,0,0,0.6)';
    ctx.fillRect(0, 0, W, H);
    ctx.fillStyle = '#fff';
    ctx.font = 'bold 40px Arial';
    ctx.textAlign = 'center';
    ctx.fillText('Игра окончена', W / 2, H / 2 - 20);
    ctx.font = '24px Arial';
    ctx.fillText('Очки: ' + Math.floor(score), W / 2, H / 2 + 20);
  }
}

// --- Перезапуск ---
restartBtn.addEventListener('click', () => {
  plane.x = W / 2 - plane.w / 2;
  plane.y = H - 100;
  plane.cooldown = 0;
  bullets = [];
  enemies = [];
  particles = [];
  enemyTimer = 0;
  enemyInterval = 80;
  score = 0;
  scoreEl.textContent = 0;
  gameOver = false;
  restartBtn.style.display = 'none';
});

// --- Игровой цикл ---
function loop() {
  update();
  draw();
  requestAnimationFrame(loop);
}
loop();
