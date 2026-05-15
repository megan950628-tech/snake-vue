<template>
  <div class="game-container">
    <div class="header">
      <h1>Score: {{ score }}</h1>
      <h2 class="level-tag">Level: {{ level }}</h2>
    </div>
    
    <div class="canvas-wrapper">
      <canvas ref="gameCanvas" width="400" height="400" class="canvas"></canvas>

      <div v-if="!gameStarted || gameOver" class="overlay">
        <h2 v-if="gameOver">Game Over!</h2>
        <h2 v-else>Snake Master</h2>
        <p v-if="gameOver">Final Score: {{ score }} (Lv.{{ level }})</p>
        <button @click="startGame" class="start-btn">Start</button>
      </div>
    </div>
  </div>
</template>
<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

const gameCanvas = ref(null);
const score = ref(0);
const gameOver = ref(false);
const gameStarted = ref(false); // 新增：控制遊戲是否啟動
const grid = 20;

// 在 script setup 中新增/修改
const level = ref(1);
const speed = ref(150); // 初始速度：150ms
const fruits = [
  { color: '#ff7675', score: 10 }, // 蘋果
  { color: '#ffeaa7', score: 20 }, // 檸檬
  { color: '#a29bfe', score: 30 }  // 葡萄
];
let currentFruit = fruits[0]; // 目前畫面上水果的屬性

let ctx = null;
let gameInterval = null;
let snake = [];
const foods = ref([]); // 現在儲存多個水果
const maxFoodCount = ref(1);
let direction = { x: grid, y: 0 };

const createFood = () => {
  // 補足水果直到達到當前等級允許的上限
  while (foods.value.length < maxFoodCount.value) {
    const newFood = {
      x: Math.floor(Math.random() * 20) * grid,
      y: Math.floor(Math.random() * 20) * grid,
      // 隨機分配這顆水果的種類
      type: fruits[Math.floor(Math.random() * fruits.length)]
    };
    
    // 簡單檢查：避免水果疊在一起或生在蛇身上
    const isOccupied = snake.some(p => p.x === newFood.x && p.y === newFood.y) ||
                       foods.value.some(f => f.x === newFood.x && f.y === newFood.y);
    
    if (!isOccupied) {
      foods.value.push(newFood);
    }
  }
};

const checkLevelUp = () => {
  const newLevel = Math.floor(score.value / 100) + 1;
  
  if (newLevel > level.value) {
    level.value = newLevel;
    speed.value = Math.max(50, speed.value - 15);
    
    // 難度門檻：Level 3 開始有 2 顆，Level 5 開始每級增加 1 顆
    if (level.value >= 5) {
      maxFoodCount.value = level.value - 2; 
    } else if (level.value >= 3) {
      maxFoodCount.value = 2;
    }

    clearInterval(gameInterval);
    gameInterval = setInterval(gameLoop, speed.value);
    createFood(); // 升級後立即補滿水果
  }
};

// 核心：開始遊戲的功能
const startGame = () => {
  snake = [{ x: 200, y: 200 }];
  direction = { x: grid, y: 0 };
  score.value = 0;
  level.value = 1;
  speed.value = 150;
  maxFoodCount.value = 1; // 重置數量
  foods.value = []; // 清空水果陣列
  gameOver.value = false;
  gameStarted.value = true;
  
  createFood();
  
  if (gameInterval) clearInterval(gameInterval);
  gameInterval = setInterval(gameLoop, speed.value);
};

const gameLoop = () => {
  moveSnake();
  draw();
};

const moveSnake = () => {
  const head = { x: snake[0].x + direction.x, y: snake[0].y + direction.y };

  if (head.x < 0 || head.x >= 400 || head.y < 0 || head.y >= 400 || 
      snake.some(s => s.x === head.x && s.y === head.y)) {
    return endGame();
  }

  snake.unshift(head);

  // 檢查是否吃到陣列中的任一水果
  const foodIndex = foods.value.findIndex(f => f.x === head.x && f.y === head.y);

  if (foodIndex !== -1) {
    // 吃到水果了！
    score.value += foods.value[foodIndex].type.score;
    foods.value.splice(foodIndex, 1); // 移除被吃掉的那顆
    checkLevelUp();
    createFood(); // 補滿水果
  } else {
    snake.pop();
  }
};

const draw = () => {
  ctx.fillStyle = '#000';
  ctx.fillRect(0, 0, 400, 400);


  // 2. 繪製背景網格 (新增部分)
  ctx.strokeStyle = '#2c3e50'; // 網格顏色，深灰色不搶戲
  ctx.lineWidth = 1;

  for (let i = 0; i <= 400; i += grid) {
    // 畫縱線
    ctx.beginPath();
    ctx.moveTo(i, 0);
    ctx.lineTo(i, 400);
    ctx.stroke();

    // 畫橫線
    ctx.beginPath();
    ctx.moveTo(0, i);
    ctx.lineTo(400, i);
    ctx.stroke();

  // 繪製所有水果
  foods.value.forEach(f => {
    ctx.fillStyle = f.type.color;
    ctx.shadowBlur = 8;
    ctx.shadowColor = f.type.color;
    ctx.beginPath();
    ctx.arc(f.x + grid/2, f.y + grid/2, grid/2 - 2, 0, Math.PI * 2);
    ctx.fill();
  });
  ctx.shadowBlur = 0; // 重設發光

  // 繪製蛇
  snake.forEach((part, i) => {
    ctx.fillStyle = i === 0 ? '#55efc4' : '#00b894';
    ctx.fillRect(part.x + 1, part.y + 1, grid - 2, grid - 2);
  });
};
};

const endGame = () => {
  gameOver.value = true;
  clearInterval(gameInterval);
};

const handleKey = (e) => {
  // 只有在遊戲開始後才偵測方向
  if (!gameStarted.value) return; 
  
  switch (e.key) {
    case 'ArrowUp': if (direction.y === 0) direction = { x: 0, y: -grid }; break;
    case 'ArrowDown': if (direction.y === 0) direction = { x: 0, y: grid }; break;
    case 'ArrowLeft': if (direction.x === 0) direction = { x: -grid, y: 0 }; break;
    case 'ArrowRight': if (direction.x === 0) direction = { x: grid, y: 0 }; break;
  }
};

onMounted(() => {
  ctx = gameCanvas.value.getContext('2d');
  // 先畫一次黑底畫布，讓介面好看
  ctx.fillStyle = '#000';
  ctx.fillRect(0, 0, 400, 400);
  window.addEventListener('keydown', handleKey);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKey);
  clearInterval(gameInterval);
});
</script>
<style scoped>
.game-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-top: 50px;
}

.canvas-wrapper {
  position: relative; /* 讓 overlay 可以相對於此定位 */
}

.canvas {
  border: 4px solid #34495e;
  background-color: #000;
  display: block;
}

.overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.75);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  color: white;
}

.start-btn {
  padding: 12px 30px;
  font-size: 1.2rem;
  background-color: #55efc4;
  border: none;
  border-radius: 25px;
  cursor: pointer;
  font-weight: bold;
  transition: transform 0.2s;
}

.start-btn:hover {
  transform: scale(1.1);
  background-color: #00b894;
}

.final-score {
  font-size: 1.5rem;
  margin-bottom: 20px;
  color: #fab1a0;
}
</style>