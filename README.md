const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

const overlay = document.getElementById('overlay');
const overlayTitle = document.getElementById('overlayTitle');
const overlayText = document.getElementById('overlayText');
const startButton = document.getElementById('startButton');
const resetButton = document.getElementById('resetButton');
const leftScoreEl = document.getElementById('leftScore');
const rightScoreEl = document.getElementById('rightScore');

const state = {
  width: canvas.width,
  height: canvas.height,
  leftScore: 0,
  rightScore: 0,
  running: false,
  matchOver: false,
  serving: 'left',
  lastTime: 0,
  difficulty: 0.18,
};

const leftPaddle = {
  x: 42,
  y: canvas.height / 2 - 62,
  width: 16,
  height: 124,
  speed: 7.5,
  direction: 0,
};

const rightPaddle = {
  x: canvas.width - 58,
  y: canvas.height / 2 - 62,
  width: 16,
  height: 124,
  speed: 6.2,
};

const ball = {
  x: canvas.width / 2,
  y: canvas.height / 2,
  radius: 10,
  speed: 7,
  vx: 5,
  vy: 3,
};

const keys = {
  w: false,
  s: false,
  ArrowUp: false,
  ArrowDown: false,
};

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

function setOverlay(title, text, showButton = true) {
  overlayTitle.textContent = title;
  overlayText.textContent = text;
  overlay.classList.remove('hidden');
  startButton.style.display = showButton ? 'inline-flex' : 'none';
}

function updateScoreboard() {
  leftScoreEl.textContent = state.leftScore;
  rightScoreEl.textContent = state.rightScore;
}

function resetBall(servingSide) {
  ball.x = canvas.width / 2;
  ball.y = canvas.height / 2;
  ball.speed = 7;

  const angle = (Math.random() * 0.8 - 0.4) * Math.PI * 0.8;
  const direction = servingSide === 'left' ? 1 : -1;
  ball.vx = direction * (Math.cos(angle) * 7 + 2.5);
  ball.vy = Math.sin(angle) * 7;

  if (Math.abs(ball.vx) < 4.2) {
    ball.vx = direction * 5.4;
  }

  if (Math.abs(ball.vy) < 2.8) {
    ball.vy = (Math.random() > 0.5 ? 1 : -1) * 3.2;
  }

  state.serving = servingSide;
  state.running = false;
  setOverlay(
    servingSide === 'left' ? 'Your serve' : 'CPU serve',
    'Press Space to launch the rally.',
    true
  );
}

function resetMatch() {
  state.leftScore = 0;
  state.rightScore = 0;
  state.matchOver = false;
  updateScoreboard();
  resetBall('left');
}

function startRound() {
  if (state.matchOver) return;
  state.running = true;
  overlay.classList.add('hidden');
}

function scorePoint(side) {
  if (side === 'left') {
    state.leftScore += 1;
  } else {
    state.rightScore += 1;
  }

  updateScoreboard();

  if (state.leftScore >= 7 || state.rightScore >= 7) {
    state.matchOver = true;
    state.running = false;
    const winner = state.leftScore > state.rightScore ? 'You win!' : 'CPU wins!';
    setOverlay(winner, 'Tap New Match to play again.', true);
    return;
  }

  resetBall(side === 'left' ? 'right' : 'left');
}

function updatePaddles() {
  if (keys.w) leftPaddle.y -= leftPaddle.speed;
  if (keys.s) leftPaddle.y += leftPaddle.speed;

  if (keys.ArrowUp) rightPaddle.y -= rightPaddle.speed;
  if (keys.ArrowDown) rightPaddle.y += rightPaddle.speed;

  leftPaddle.y = clamp(leftPaddle.y, 12, state.height - leftPaddle.height - 12);
  rightPaddle.y = clamp(rightPaddle.y, 12, state.height - rightPaddle.height - 12);

  if (!state.running) return;

  const targetY = ball.y - rightPaddle.height / 2;
  const aiSpeed = rightPaddle.speed + state.difficulty * 6;
  rightPaddle.y += (targetY - rightPaddle.y) * state.difficulty * 1.4 + (ball.vy * 0.12);
  rightPaddle.y = clamp(rightPaddle.y, 12, state.height - rightPaddle.height - 12);
}

function handleBallPaddleCollision(paddle) {
  const paddleLeft = paddle.x;
  const paddleRight = paddle.x + paddle.width;
  const paddleTop = paddle.y;
  const paddleBottom = paddle.y + paddle.height;

  const overlapsX = ball.x + ball.radius > paddleLeft && ball.x - ball.radius < paddleRight;
  const overlapsY = ball.y + ball.radius > paddleTop && ball.y - ball.radius < paddleBottom;

  if (!overlapsX || !overlapsY) return false;

  const relativeIntersect = (ball.y - (paddle.y + paddle.height / 2)) / (paddle.height / 2);
  const bounceAngle = relativeIntersect * (Math.PI / 3);

  const speedBoost = clamp(Math.abs(ball.vx) * 0.12 + 0.7, 0.8, 3);
  const xDirection = ball.x < canvas.width / 2 ? 1 : -1;

  ball.vx = xDirection * (Math.cos(bounceAngle) * (ball.speed + speedBoost));
  ball.vy = Math.sin(bounceAngle) * (ball.speed + speedBoost * 0.55);
  ball.speed = clamp(Math.hypot(ball.vx, ball.vy) + 0.35, 7.5, 15);

  ball.x = ball.x < canvas.width / 2 ? paddleLeft - ball.radius : paddleRight + ball.radius;
  return true;
}

function updateBall() {
  if (!state.running) return;

  ball.x += ball.vx;
  ball.y += ball.vy;

  if (ball.y - ball.radius <= 12 || ball.y + ball.radius >= canvas.height - 12) {
    ball.vy *= -1;
    ball.y = clamp(ball.y, ball.radius + 12, canvas.height - ball.radius - 12);
  }

  if (handleBallPaddleCollision(leftPaddle) || handleBallPaddleCollision(rightPaddle)) {
    return;
  }

  if (ball.x < 0) {
    scorePoint('right');
    return;
  }

  if (ball.x > canvas.width) {
    scorePoint('left');
    return;
  }
}

function drawRoundedRect(x, y, width, height, radius, fillStyle) {
  ctx.beginPath();
  ctx.moveTo(x + radius, y);
  ctx.lineTo(x + width - radius, y);
  ctx.quadraticCurveTo(x + width, y, x + width, y + radius);
  ctx.lineTo(x + width, y + height - radius);
  ctx.quadraticCurveTo(x + width, y + height, x + width - radius, y + height);
  ctx.lineTo(x + radius, y + height);
  ctx.quadraticCurveTo(x, y + height, x, y + height - radius);
  ctx.lineTo(x, y + radius);
  ctx.quadraticCurveTo(x, y, x + radius, y);
  ctx.closePath();
  ctx.fillStyle = fillStyle;
  ctx.fill();
}

function drawCourt() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  const courtGradient = ctx.createLinearGradient(0, 0, 0, canvas.height);
  courtGradient.addColorStop(0, '#0d5d4f');
  courtGradient.addColorStop(1, '#0b473f');
  ctx.fillStyle = courtGradient;
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  ctx.fillStyle = 'rgba(255,255,255,0.04)';
  for (let i = 0; i < 16; i += 1) {
    const x = (i / 16) * canvas.width;
    ctx.fillRect(x, 0, 1, canvas.height);
  }

  ctx.strokeStyle = 'rgba(234, 241, 255, 0.75)';
  ctx.lineWidth = 4;
  ctx.setLineDash([10, 14]);
  ctx.beginPath();
  ctx.moveTo(canvas.width / 2, 12);
  ctx.lineTo(canvas.width / 2, canvas.height - 12);
  ctx.stroke();
  ctx.setLineDash([]);

  ctx.strokeStyle = 'rgba(234, 241, 255, 0.7)';
  ctx.lineWidth = 3;
  ctx.strokeRect(18, 12, canvas.width - 36, canvas.height - 24);

  ctx.strokeRect(canvas.width / 2 - 130, 12, 260, canvas.height - 24);

  ctx.beginPath();
  ctx.arc(canvas.width / 2, canvas.height / 2, 72, 0, Math.PI * 2);
  ctx.stroke();

  const netGlow = ctx.createLinearGradient(0, 0, canvas.width, 0);
  netGlow.addColorStop(0, 'rgba(169,255,221,0.2)');
  netGlow.addColorStop(0.5, 'rgba(123,227,255,0.6)');
  netGlow.addColorStop(1, 'rgba(169,255,221,0.2)');
  ctx.strokeStyle = netGlow;
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.moveTo(canvas.width / 2, 12);
  ctx.lineTo(canvas.width / 2, canvas.height - 12);
  ctx.stroke();
}

function drawBall() {
  const fill = ctx.createRadialGradient(
    ball.x - 3,
    ball.y - 4,
    2,
    ball.x,
    ball.y,
    ball.radius + 8
  );
  fill.addColorStop(0, '#fffdf4');
  fill.addColorStop(0.35, '#f7f0cd');
  fill.addColorStop(1, '#ffc857');

  ctx.beginPath();
  ctx.arc(ball.x, ball.y, ball.radius, 0, Math.PI * 2);
  ctx.fillStyle = fill;
  ctx.fill();

  ctx.beginPath();
  ctx.arc(ball.x - 3, ball.y - 4, ball.radius * 0.35, 0, Math.PI * 2);
  ctx.fillStyle = 'rgba(255,255,255,0.3)';
  ctx.fill();
}

function drawPaddle(paddle, fillColor) {
  drawRoundedRect(paddle.x, paddle.y, paddle.width, paddle.height, 14, fillColor);

  ctx.fillStyle = 'rgba(255,255,255,0.2)';
  ctx.fillRect(paddle.x + 5, paddle.y + 18, paddle.width - 10, paddle.height - 36);
}

function drawTrail() {
  const trailCount = 12;
  for (let i = 0; i < trailCount; i += 1) {
    const t = (i + 1) / trailCount;
    const alpha = 0.12 * (1 - t);
    const x = ball.x - ball.vx * i * 1.2;
    const y = ball.y - ball.vy * i * 1.2;
    ctx.beginPath();
    ctx.arc(x, y, ball.radius * (0.55 - t * 0.2), 0, Math.PI * 2);
    ctx.fillStyle = `rgba(255, 200, 87, ${alpha})`;
    ctx.fill();
  }
}

function draw() {
  drawCourt();
  drawTrail();
  drawPaddle(leftPaddle, 'rgba(123, 227, 255, 0.96)');
  drawPaddle(rightPaddle, 'rgba(255, 200, 87, 0.96)');
  drawBall();
}

function gameLoop(ts) {
  const delta = (ts - state.lastTime) / 16.67 || 1;
  state.lastTime = ts;

  updatePaddles();
  updateBall();
  draw();

  requestAnimationFrame(gameLoop);
}

window.addEventListener('keydown', (event) => {
  const key = event.key;
  if (key === ' ' || key === 'Spacebar') {
    event.preventDefault();
    if (!state.running && !state.matchOver) {
      startRound();
    }
  }

  if (key === 'w' || key === 'W') keys.w = true;
  if (key === 's' || key === 'S') keys.s = true;
  if (key === 'ArrowUp') keys.ArrowUp = true;
  if (key === 'ArrowDown') keys.ArrowDown = true;
});

window.addEventListener('keyup', (event) => {
  const key = event.key;
  if (key === 'w' || key === 'W') keys.w = false;
  if (key === 's' || key === 'S') keys.s = false;
  if (key === 'ArrowUp') keys.ArrowUp = false;
  if (key === 'ArrowDown') keys.ArrowDown = false;
});

startButton.addEventListener('click', () => {
  if (state.matchOver) {
    resetMatch();
  }
  startRound();
});

resetButton.addEventListener('click', () => {
  resetMatch();
});

resetMatch();
requestAnimationFrame(gameLoop);
