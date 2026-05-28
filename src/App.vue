<template>
  <div class="game-wrapper-centered">
    <div class="game-container">
      <div class="game-header">
        <div class="header-left">
          <h2 class="info-text">Score: {{ score }}</h2>
          <h2 class="info-text level-color">Level: {{ level }}</h2>
        </div>

        <div class="header-center">
          <div class="status-badge">
            <span v-if="isInvincible" class="badge invincible">
              ⚠️ 無敵逃跑中！({{ invincibleTimeLeft.toFixed(2) }}s)
            </span>
            <span v-else-if="rottenBodies.length > 0" class="badge danger">
              ⚡ 殘骸阻擋中！({{ rottenTimeLeft.toFixed(1) }}s)
            </span>
            <span v-else class="badge safe">🟢 戰場安全</span>
          </div>
        </div>

        <div class="header-right">
          <h2 class="info-text best-color">Best: {{ bestScore }}</h2>
        </div>
      </div>
      
      <div class="canvas-wrapper">
        <canvas ref="gameCanvas" width="400" height="400" class="canvas"></canvas>

        <div v-if="!gameStarted || gameOver" class="overlay">
          <h2 v-if="gameOver" class="game-over-title">Game Over!</h2>
          <h2 v-else class="game-title">Snake Master</h2>
          <p v-if="gameOver" class="final-score">Final Score: {{ score }} (Lv.{{ level }})</p>
          <button @click="startGame" class="start-btn">Start</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

const gameCanvas = ref(null);
const score = ref(0);
const bestScore = ref(0); 
const gameOver = ref(false);
const gameStarted = ref(false); 
const grid = 20;

const level = ref(1);
const speed = ref(150); 
const fruits = [
  { name: 'apple', color: '#ff7675', score: 10 },  
  { name: 'lemon', color: '#ffeaa7', score: 20 },  
  { name: 'grape', color: '#a29bfe', score: 30 }   
];

let ctx = null;
let gameInterval = null;
let snake = [];
const foods = ref([]); 
const bombs = ref([]); 
const maxFoodCount = ref(1);
let direction = { x: grid, y: 0 };

const rottenBodies = ref([]);      
const isInvincible = ref(false);   
const invincibleTimeLeft = ref(0); 
const rottenTimeLeft = ref(0);     

let rottenCountdownId = null;      
let invincibleCountdownId = null;  

const getRandomPosition = () => {
  return {
    x: Math.floor(Math.random() * 20) * grid,
    y: Math.floor(Math.random() * 20) * grid
  };
};

const isPosOccupied = (x, y) => {
  return snake.some(p => p.x === x && p.y === y) ||
         foods.value.some(f => f.x === x && f.y === y) ||
         bombs.value.some(b => b.x === x && b.y === y) ||
         rottenBodies.value.some(r => r.x === x && r.y === y);
};

const createObjects = () => {
  // 1. 補足水果
  while (foods.value.length < maxFoodCount.value) {
    const pos = getRandomPosition();
    if (!isPosOccupied(pos.x, pos.y)) {
      const foodItem = {
        x: pos.x,
        y: pos.y,
        type: fruits[Math.floor(Math.random() * fruits.length)],
        // 🌟 已修改：生存計時器改為 10 秒鐘（10000ms）
        timerId: setTimeout(() => {
          refreshSingleFood(foodItem);
        }, 10000)
      };
      foods.value.push(foodItem);
    }
  }

  // 2. 補足炸彈
  while (bombs.value.length < maxFoodCount.value) {
    const pos = getRandomPosition();
    if (!isPosOccupied(pos.x, pos.y)) {
      const bombItem = {
        x: pos.x,
        y: pos.y,
        // 🌟 已修改：生存計時器改為 10 秒鐘（10000ms）
        timerId: setTimeout(() => {
          refreshSingleBomb(bombItem);
        }, 10000)
      };
      bombs.value.push(bombItem);
    }
  }
};

const refreshSingleFood = (foodItem) => {
  const index = foods.value.indexOf(foodItem);
  if (index !== -1) {
    clearTimeout(foodItem.timerId); 
    foods.value.splice(index, 1);
    createObjects(); 
    draw(); 
  }
};

const refreshSingleBomb = (bombItem) => {
  const index = bombs.value.indexOf(bombItem);
  if (index !== -1) {
    clearTimeout(bombItem.timerId);
    bombs.value.splice(index, 1);
    createObjects();
    draw();
  }
};

const clearAllObjectTimers = () => {
  foods.value.forEach(f => { if (f.timerId) clearTimeout(f.timerId); });
  bombs.value.forEach(b => { if (b.timerId) clearTimeout(b.timerId); });
  foods.value = [];
  bombs.value = [];
};

const checkLevelUp = () => {
  const newLevel = Math.floor(score.value / 100) + 1;
  if (newLevel > level.value) {
    level.value = newLevel;
    speed.value = Math.max(50, speed.value - 15);
    
    if (level.value >= 5) {
      maxFoodCount.value = level.value - 2; 
    } else if (level.value >= 3) {
      maxFoodCount.value = 2;
    }

    clearInterval(gameInterval);
    gameInterval = setInterval(gameLoop, speed.value);
    
    // 升級時重置所有物件，重新起算 10 秒
    clearAllObjectTimers();
    createObjects();
  }
};

const startGame = () => {
  if (rottenCountdownId) clearInterval(rottenCountdownId);
  if (invincibleCountdownId) clearInterval(invincibleCountdownId);
  clearAllObjectTimers(); 

  snake = [{ x: 200, y: 200 }, { x: 180, y: 200 }, { x: 160, y: 200 }]; 
  direction = { x: grid, y: 0 };
  score.value = 0;
  level.value = 1;
  speed.value = 150;
  maxFoodCount.value = 1; 
  rottenBodies.value = [];
  isInvincible.value = false;
  invincibleTimeLeft.value = 0;
  rottenTimeLeft.value = 0;
  gameOver.value = false;
  gameStarted.value = true;
  
  createObjects();
  
  if (gameInterval) clearInterval(gameInterval);
  gameInterval = setInterval(gameLoop, speed.value);
};

const gameLoop = () => {
  moveSnake();
  draw();
};

const startInvincibleCountdown = () => {
  isInvincible.value = true;
  invincibleTimeLeft.value = 1.00; 

  if (invincibleCountdownId) clearInterval(invincibleCountdownId);

  invincibleCountdownId = setInterval(() => {
    invincibleTimeLeft.value -= 0.01;
    if (invincibleTimeLeft.value <= 0) {
      invincibleTimeLeft.value = 0;
      isInvincible.value = false;
      clearInterval(invincibleCountdownId);
    }
  }, 10);
};

const startRottenCountdown = () => {
  rottenTimeLeft.value = 10.0;

  if (rottenCountdownId) clearInterval(rottenCountdownId);

  rottenCountdownId = setInterval(() => {
    rottenTimeLeft.value -= 0.1;
    if (rottenTimeLeft.value <= 0) {
      rottenTimeLeft.value = 0;
      rottenBodies.value = []; 
      clearInterval(rottenCountdownId);
    }
  }, 100);
};

const moveSnake = () => {
  let nextX = snake[0].x + direction.x;
  let nextY = snake[0].y + direction.y;

  if (nextX < 0) nextX = 400 - grid;
  if (nextX >= 400) nextX = 0;
  if (nextY < 0) nextY = 400 - grid;
  if (nextY >= 400) nextY = 0;

  const head = { x: nextX, y: nextY };

  if (bombs.value.some(b => b.x === head.x && b.y === head.y)) {
    return endGame();
  }

  if (!isInvincible.value) {
    if (rottenBodies.value.some(r => r.x === head.x && r.y === head.y)) {
      return endGame();
    }

    const hitIndex = snake.findIndex(s => s.x === head.x && s.y === head.y);
    if (hitIndex !== -1) {
      const brokenPart = snake.splice(hitIndex);
      rottenBodies.value.push(...brokenPart);

      score.value = Math.max(0, score.value - brokenPart.length * 5);

      startInvincibleCountdown();
      startRottenCountdown(); 
    }
  }

  if (snake.length === 0) {
    snake.unshift(head);
  } else {
    snake.unshift(head);
  }

  const foodIndex = foods.value.findIndex(f => f.x === head.x && f.y === head.y);

  if (foodIndex !== -1) {
    clearTimeout(foods.value[foodIndex].timerId); // 吃到時記得清除 10 秒計時器
    
    score.value += foods.value[foodIndex].type.score;
    
    if (score.value > bestScore.value) {
      bestScore.value = score.value;
      localStorage.setItem('snake_best_score', bestScore.value); 
    }

    foods.value.splice(foodIndex, 1); 
    checkLevelUp();
    createObjects(); 
  } else {
    snake.pop();
  }
};

const drawFruit = (ctx, f) => {
  const cx = f.x + grid / 2;
  const cy = f.y + grid / 2;
  const r = grid / 2 - 2;
  ctx.save();
  
  if (f.type.name === 'apple') {
    ctx.fillStyle = '#ff7675';
    ctx.beginPath(); ctx.moveTo(cx, cy - r + 3);
    ctx.bezierCurveTo(cx - r, cy - r, cx - r, cy + r - 2, cx, cy + r - 1);
    ctx.bezierCurveTo(cx + r, cy + r - 2, cx + r, cy - r, cx, cy - r + 3); ctx.fill();
    ctx.strokeStyle = '#e17055'; ctx.lineWidth = 2;
    ctx.beginPath(); ctx.moveTo(cx, cy - r + 3); ctx.quadraticCurveTo(cx + 3, cy - r - 3, cx + 5, cy - r - 4); ctx.stroke();
    ctx.fillStyle = '#2ecc71';
    ctx.beginPath(); ctx.ellipse(cx + 3, cy - r - 2, 3, 1.5, Math.PI / 4, 0, Math.PI * 2); ctx.fill();
  } else if (f.type.name === 'lemon') {
    ctx.fillStyle = '#ffeaa7';
    ctx.beginPath(); ctx.ellipse(cx, cy, r, r - 3, Math.PI / 6, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#fdcb6e';
    ctx.beginPath(); ctx.arc(cx - r + 1, cy - 1, 1.5, 0, Math.PI * 2); ctx.arc(cx + r - 1, cy + 1, 1.5, 0, Math.PI * 2); ctx.fill();
  } else if (f.type.name === 'grape') {
    ctx.fillStyle = '#a29bfe';
    ctx.beginPath(); ctx.arc(cx - 3, cy - 2, 4, 0, Math.PI * 2); ctx.fill();
    ctx.beginPath(); ctx.arc(cx + 3, cy - 2, 4, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#6c5ce7'; 
    ctx.beginPath(); ctx.arc(cx, cy + 3, 4, 0, Math.PI * 2); ctx.fill();
    ctx.strokeStyle = '#2ecc71'; ctx.lineWidth = 1.5;
    ctx.beginPath(); ctx.moveTo(cx, cy - 4); ctx.quadraticCurveTo(cx - 2, cy - 7, cx, cy - 8); ctx.stroke();
  }
  ctx.restore();
};

const drawBomb = (ctx, b) => {
  const cx = b.x + grid / 2;
  const cy = b.y + grid / 2;
  const r = grid / 2 - 3;
  ctx.save();

  ctx.strokeStyle = '#95a5a6';
  ctx.lineWidth = 1.5;
  ctx.beginPath();
  ctx.moveTo(cx, cy - r);
  ctx.quadraticCurveTo(cx + 4, cy - r - 4, cx + 6, cy - r - 5);
  ctx.stroke();

  ctx.fillStyle = '#ff7675'; 
  ctx.beginPath(); ctx.arc(cx + 6, cy - r - 5, 2, 0, Math.PI * 2); ctx.fill();
  ctx.fillStyle = '#ffeaa7'; 
  ctx.beginPath(); ctx.arc(cx + 6, cy - r - 5, 1, 0, Math.PI * 2); ctx.fill();

  // 💣 炸彈本體：鋼鐵灰色
  ctx.fillStyle = '#7f8c8d'; 
  ctx.beginPath();
  ctx.arc(cx, cy, r, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#2c3e50';
  ctx.fillRect(cx - 2, cy - r - 1, 4, 2);

  ctx.fillStyle = 'rgba(255, 255, 255, 0.5)'; 
  ctx.beginPath();
  ctx.arc(cx - 2, cy - 2, 1.2, 0, Math.PI * 2);
  ctx.fill();

  ctx.restore();
};

const draw = () => {
  ctx.fillStyle = '#1a252f'; 
  ctx.fillRect(0, 0, 400, 400);

  ctx.strokeStyle = '#2c3e50'; 
  ctx.lineWidth = 1;
  for (let i = 0; i <= 400; i += grid) {
    ctx.beginPath(); ctx.moveTo(i, 0); ctx.lineTo(i, 400); ctx.stroke();
    ctx.beginPath(); ctx.moveTo(0, i); ctx.lineTo(400, i); ctx.stroke();
  }

  foods.value.forEach(f => drawFruit(ctx, f));
  bombs.value.forEach(b => drawBomb(ctx, b));

  rottenBodies.value.forEach(r => {
    ctx.fillStyle = '#636e72'; 
    ctx.beginPath();
    ctx.arc(r.x + grid / 2, r.y + grid / 2, grid / 2 - 2, 0, Math.PI * 2);
    ctx.fill();
  });

  snake.forEach((part, i) => {
    const cx = part.x + grid / 2;
    const cy = part.y + grid / 2;

    if (isInvincible.value) {
      const shouldBlink = Math.floor(invincibleTimeLeft.value * 10) % 2 === 0;
      ctx.fillStyle = shouldBlink ? 'rgba(85, 239, 196, 0.3)' : 'rgba(85, 239, 196, 0.7)';
    } else {
      ctx.fillStyle = '#55efc4'; 
    }

    if (i === 0) {
      ctx.beginPath(); ctx.arc(cx, cy, grid / 2, 0, Math.PI * 2); ctx.fill();
      ctx.fillStyle = '#ffffff';
      let eyeX1 = 0, eyeY1 = 0, eyeX2 = 0, eyeY2 = 0;
      if (direction.x > 0) { eyeX1 = cx + 4; eyeY1 = cy - 4; eyeX2 = cx + 4; eyeY2 = cy + 4; } 
      else if (direction.x < 0) { eyeX1 = cx - 4; eyeY1 = cy - 4; eyeX2 = cx - 4; eyeY2 = cy + 4; } 
      else if (direction.y > 0) { eyeX1 = cx - 4; eyeY1 = cy + 4; eyeX2 = cx + 4; eyeY2 = cy + 4; } 
      else if (direction.y < 0) { eyeX1 = cx - 4; eyeY1 = cy - 4; eyeX2 = cx + 4; eyeY2 = cy - 4; }
      ctx.beginPath(); ctx.arc(eyeX1, eyeY1, 3, 0, Math.PI * 2); ctx.fill();
      ctx.beginPath(); ctx.arc(eyeX2, eyeY2, 3, 0, Math.PI * 2); ctx.fill();
      ctx.fillStyle = '#000000';
      ctx.beginPath(); ctx.arc(eyeX1, eyeY1, 1.5, 0, Math.PI * 2); ctx.fill();
      ctx.beginPath(); ctx.arc(eyeX2, eyeY2, 1.5, 0, Math.PI * 2); ctx.fill();
    } else {
      ctx.beginPath();
      ctx.arc(cx, cy, grid / 2 - 2, 0, Math.PI * 2);
      ctx.fill();
    }
  });
};

const endGame = () => {
  gameOver.value = true;
  clearInterval(gameInterval);
  if (invincibleCountdownId) clearInterval(invincibleCountdownId);
  if (rottenCountdownId) clearInterval(rottenCountdownId);
  clearAllObjectTimers(); 
};

const handleKey = (e) => {
  if (!gameStarted.value) return; 
  switch (e.key) {
    case 'ArrowUp': if (direction.y === 0) direction = { x: 0, y: -grid }; break;
    case 'ArrowDown': if (direction.y === 0) direction = { x: 0, y: grid }; break;
    case 'ArrowLeft': if (direction.x === 0) direction = { x: -grid, y: 0 }; break;
    case 'ArrowRight': if (direction.x === 0) direction = { x: grid, y: 0 }; break;
  }
};

onMounted(() => {
  const savedBestScore = localStorage.getItem('snake_best_score');
  if (savedBestScore) {
    bestScore.value = parseInt(savedBestScore, 10);
  }

  ctx = gameCanvas.value.getContext('2d');
  ctx.fillStyle = '#1a252f';
  ctx.fillRect(0, 0, 400, 400);
  window.addEventListener('keydown', handleKey);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKey);
  clearInterval(gameInterval);
  if (rottenCountdownId) clearInterval(rottenCountdownId);
  if (invincibleCountdownId) clearInterval(invincibleCountdownId);
  clearAllObjectTimers();
});
</script>

<style scoped>
.game-wrapper-centered {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh; 
  background-color: #f5f6fa; 
  margin: 0;
  box-sizing: border-box;
}

.game-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  width: 412px; 
}

.game-header {
  display: flex;
  justify-content: space-between;
  align-items: center; 
  width: 100%;
  margin-bottom: 12px;
}

.header-left {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  flex: 1;
}

.header-center {
  display: flex;
  justify-content: center;
  align-items: center;
}

.header-right {
  display: flex;
  justify-content: flex-end;
  align-items: center;
  flex: 1;
}

.info-text {
  margin: 0;
  font-size: 1.2rem; 
  font-weight: 700;
  color: #2d3436;
  line-height: 1.3;
}

.level-color { color: #0984e3; }
.best-color { color: #e67e22; }

.status-badge {
  display: flex;
  align-items: center;
  height: 30px;
}

.badge {
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 0.85rem;
  font-weight: bold;
  white-space: nowrap; 
}
.badge.safe { background-color: #dfe6e9; color: #636e72; }
.badge.invincible { background-color: #ffeaa7; color: #d63031; }
.badge.danger { background-color: #ff7675; color: white; }

.canvas-wrapper {
  position: relative; 
  box-shadow: 0 15px 35px rgba(0,0,0,0.15);
  border-radius: 10px;
  overflow: hidden;
}

.canvas {
  border: 6px solid #2c3e50;
  background-color: #1a252f;
  display: block;
}

.overlay {
  position: absolute;
  top: 0; left: 0; width: 100%; height: 100%;
  background: rgba(26, 37, 47, 0.85);
  display: flex; flex-direction: column; justify-content: center; align-items: center;
  color: white; backdrop-filter: blur(4px);
}

.game-title { font-size: 2.5rem; margin-bottom: 20px; color: #55efc4; text-shadow: 0 2px 4px rgba(0,0,0,0.5); }
.game-over-title { font-size: 2.5rem; margin-bottom: 10px; color: #ff7675; text-shadow: 0 2px 4px rgba(0,0,0,0.5); }
.final-score { font-size: 1.5rem; margin-bottom: 25px; color: #ffeaa7; }

.start-btn {
  padding: 12px 40px; font-size: 1.3rem; background-color: #55efc4; border: none; border-radius: 25px;
  cursor: pointer; font-weight: bold; color: #2c3e50; transition: all 0.2s ease; box-shadow: 0 4px 6px rgba(0,0,0,0.2);
}
.start-btn:hover { transform: scale(1.05) translateY(-2px); background-color: #00b894; color: white; }
</style>