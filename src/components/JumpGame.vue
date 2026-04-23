<template>
  <div class="jump-game">
    <div class="game-header">
      <div class="score">
        <span class="label">分数</span>
        <span class="value">{{ score }}</span>
      </div>
      <div class="best-score">
        <span class="label">最高分</span>
        <span class="value">{{ bestScore }}</span>
      </div>
    </div>

    <canvas
      ref="canvasRef"
      :width="canvasWidth"
      :height="canvasHeight"
      class="game-canvas"
      @mousedown="handleMouseDown"
      @mouseup="handleMouseUp"
      @mouseleave="handleMouseUp"
      @touchstart.prevent="handleMouseDown"
      @touchend.prevent="handleMouseUp"
    ></canvas>

    <div class="game-overlay" v-if="gameState === 'IDLE'">
      <div class="overlay-content">
        <h1>跳一跳</h1>
        <p class="hint">长按蓄力，松开跳跃</p>
        <button class="start-btn" @click="startGame">开始游戏</button>
      </div>
    </div>

    <div class="game-overlay" v-if="gameState === 'GAME_OVER'">
      <div class="overlay-content">
        <h1>游戏结束</h1>
        <p class="final-score">最终分数: {{ score }}</p>
        <p class="hint" v-if="score === bestScore && score > 0">🎉 新纪录！</p>
        <button class="restart-btn" @click="startGame">再来一局</button>
      </div>
    </div>

    <div class="charge-indicator" v-if="gameState === 'CHARGING'">
      <div class="charge-bar">
        <div class="charge-fill" :style="{ width: chargePercent + '%' }"></div>
      </div>
      <span class="charge-label">蓄力中...</span>
    </div>

    <div class="tips" v-if="gameState === 'PLAYING'">
      <p>长按蓄力 | 松开跳跃</p>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'

const canvasRef = ref(null)
const canvasWidth = 500
const canvasHeight = 600

const GRAVITY = 0.5
const MAX_CHARGE = 100
const CHARGE_SPEED = 1.5
const PLATFORM_MIN_WIDTH = 40
const PLATFORM_MAX_WIDTH = 80
const PLATFORM_HEIGHT = 20
const PLATFORM_MIN_GAP = 80
const PLATFORM_MAX_GAP = 150
const PLAYER_SIZE = 20

const gameState = ref('IDLE')
const score = ref(0)
const bestScore = ref(0)
const chargeTime = ref(0)
const chargePercent = computed(() => Math.min(100, (chargeTime.value / MAX_CHARGE) * 100))

let ctx = null
let animationId = null
let chargeInterval = null

const platforms = ref([])
const player = ref({
  x: 0,
  y: 0,
  vx: 0,
  vy: 0,
  isJumping: false,
  currentPlatform: 0
})

const platformColors = [
  '#FF6B6B', '#4ECDC4', '#45B7D1', '#96CEB4',
  '#FFEAA7', '#DDA0DD', '#98D8C8', '#F7DC6F',
  '#BB8FCE', '#85C1E9', '#F8B739', '#5DADE2'
]

function initGame() {
  platforms.value = []
  
  const firstPlatform = {
    x: canvasWidth / 2 - 60,
    y: canvasHeight - 150,
    width: 120,
    height: PLATFORM_HEIGHT,
    color: platformColors[0]
  }
  platforms.value.push(firstPlatform)
  
  for (let i = 1; i < 10; i++) {
    generatePlatform(i)
  }
  
  player.value = {
    x: canvasWidth / 2,
    y: firstPlatform.y - PLAYER_SIZE,
    vx: 0,
    vy: 0,
    isJumping: false,
    currentPlatform: 0
  }
  
  score.value = 0
  chargeTime.value = 0
}

function generatePlatform(index) {
  const prevPlatform = platforms.value[index - 1]
  const gap = PLATFORM_MIN_GAP + Math.random() * (PLATFORM_MAX_GAP - PLATFORM_MIN_GAP)
  const width = PLATFORM_MIN_WIDTH + Math.random() * (PLATFORM_MAX_WIDTH - PLATFORM_MIN_WIDTH)
  
  const direction = Math.random() > 0.5 ? 1 : -1
  const xOffset = (Math.random() * 50 + 30) * direction
  
  let newX = prevPlatform.x + xOffset
  newX = Math.max(20, Math.min(canvasWidth - width - 20, newX))
  
  platforms.value.push({
    x: newX,
    y: prevPlatform.y - gap,
    width: width,
    height: PLATFORM_HEIGHT,
    color: platformColors[index % platformColors.length]
  })
}

function startGame() {
  initGame()
  gameState.value = 'PLAYING'
  startRender()
}

function startRender() {
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
  render()
}

function render() {
  if (!ctx) return
  
  ctx.fillStyle = '#2C3E50'
  ctx.fillRect(0, 0, canvasWidth, canvasHeight)
  
  drawBackground()
  drawPlatforms()
  drawPlayer()
  
  animationId = requestAnimationFrame(render)
}

function drawBackground() {
  ctx.strokeStyle = 'rgba(255, 255, 255, 0.05)'
  ctx.lineWidth = 1
  
  for (let x = 0; x < canvasWidth; x += 30) {
    ctx.beginPath()
    ctx.moveTo(x, 0)
    ctx.lineTo(x, canvasHeight)
    ctx.stroke()
  }
  
  for (let y = 0; y < canvasHeight; y += 30) {
    ctx.beginPath()
    ctx.moveTo(0, y)
    ctx.lineTo(canvasWidth, y)
    ctx.stroke()
  }
}

function drawPlatforms() {
  platforms.value.forEach((platform, index) => {
    const isCurrentPlatform = index === player.value.currentPlatform && !player.value.isJumping
    
    ctx.fillStyle = 'rgba(0, 0, 0, 0.3)'
    ctx.fillRect(platform.x + 3, platform.y + 3, platform.width, platform.height)
    
    const gradient = ctx.createLinearGradient(
      platform.x, platform.y,
      platform.x, platform.y + platform.height
    )
    gradient.addColorStop(0, platform.color)
    gradient.addColorStop(1, darkenColor(platform.color, 30))
    ctx.fillStyle = gradient
    
    ctx.fillRect(platform.x, platform.y, platform.width, platform.height)
    
    if (isCurrentPlatform) {
      ctx.strokeStyle = '#FFF'
      ctx.lineWidth = 2
      ctx.strokeRect(platform.x - 1, platform.y - 1, platform.width + 2, platform.height + 2)
    }
    
    ctx.fillStyle = 'rgba(255, 255, 255, 0.3)'
    ctx.fillRect(platform.x, platform.y, platform.width, 3)
  })
}

function darkenColor(color, amount) {
  const hex = color.replace('#', '')
  const r = Math.max(0, parseInt(hex.substring(0, 2), 16) - amount)
  const g = Math.max(0, parseInt(hex.substring(2, 4), 16) - amount)
  const b = Math.max(0, parseInt(hex.substring(4, 6), 16) - amount)
  return `#${r.toString(16).padStart(2, '0')}${g.toString(16).padStart(2, '0')}${b.toString(16).padStart(2, '0')}`
}

function drawPlayer() {
  const p = player.value
  
  ctx.fillStyle = 'rgba(0, 0, 0, 0.3)'
  ctx.beginPath()
  ctx.ellipse(p.x + 2, p.y + PLAYER_SIZE + 2, PLAYER_SIZE / 2, 3, 0, 0, Math.PI * 2)
  ctx.fill()
  
  const bodyGradient = ctx.createLinearGradient(
    p.x - PLAYER_SIZE / 2, p.y - PLAYER_SIZE,
    p.x + PLAYER_SIZE / 2, p.y
  )
  bodyGradient.addColorStop(0, '#E74C3C')
  bodyGradient.addColorStop(1, '#C0392B')
  
  ctx.fillStyle = bodyGradient
  ctx.beginPath()
  ctx.roundRect(
    p.x - PLAYER_SIZE / 2,
    p.y - PLAYER_SIZE,
    PLAYER_SIZE,
    PLAYER_SIZE,
    4
  )
  ctx.fill()
  
  ctx.fillStyle = '#FFF'
  ctx.beginPath()
  ctx.arc(p.x - 5, p.y - PLAYER_SIZE / 2 - 2, 4, 0, Math.PI * 2)
  ctx.arc(p.x + 5, p.y - PLAYER_SIZE / 2 - 2, 4, 0, Math.PI * 2)
  ctx.fill()
  
  ctx.fillStyle = '#2C3E50'
  ctx.beginPath()
  ctx.arc(p.x - 4, p.y - PLAYER_SIZE / 2 - 2, 2, 0, Math.PI * 2)
  ctx.arc(p.x + 6, p.y - PLAYER_SIZE / 2 - 2, 2, 0, Math.PI * 2)
  ctx.fill()
  
  if (gameState.value === 'CHARGING') {
    const chargeHeight = (chargePercent.value / 100) * 30
    ctx.fillStyle = `rgba(255, 255, 255, ${0.3 + (chargePercent.value / 100) * 0.5})`
    ctx.fillRect(
      p.x - PLAYER_SIZE / 2 - 8,
      p.y - PLAYER_SIZE - chargeHeight,
      4,
      chargeHeight
    )
    ctx.fillRect(
      p.x + PLAYER_SIZE / 2 + 4,
      p.y - PLAYER_SIZE - chargeHeight,
      4,
      chargeHeight
    )
  }
}

function handleMouseDown() {
  if (gameState.value !== 'PLAYING' || player.value.isJumping) return
  
  gameState.value = 'CHARGING'
  chargeTime.value = 0
  
  chargeInterval = setInterval(() => {
    if (chargeTime.value < MAX_CHARGE) {
      chargeTime.value += CHARGE_SPEED
    }
  }, 16)
}

function handleMouseUp() {
  if (gameState.value !== 'CHARGING') return
  
  if (chargeInterval) {
    clearInterval(chargeInterval)
    chargeInterval = null
  }
  
  const currentPlatform = platforms.value[player.value.currentPlatform]
  const nextPlatform = platforms.value[player.value.currentPlatform + 1]
  
  if (!nextPlatform) {
    gameOver()
    return
  }
  
  const direction = nextPlatform.x > currentPlatform.x ? 1 : -1
  const distance = Math.abs(nextPlatform.x + nextPlatform.width / 2 - (currentPlatform.x + currentPlatform.width / 2))
  
  const power = chargePercent.value / 100
  const jumpVx = power * 8 * direction
  const jumpVy = -(power * 12 + 6)
  
  player.value.vx = jumpVx
  player.value.vy = jumpVy
  player.value.isJumping = true
  gameState.value = 'PLAYING'
  
  chargeTime.value = 0
  
  jumpPhysics()
}

function jumpPhysics() {
  if (!player.value.isJumping) return
  
  player.value.x += player.value.vx
  player.value.y += player.value.vy
  player.value.vy += GRAVITY
  
  checkLanding()
  
  if (player.value.isJumping) {
    requestAnimationFrame(jumpPhysics)
  }
}

function checkLanding() {
  const p = player.value
  const nextPlatformIndex = p.currentPlatform + 1
  
  if (nextPlatformIndex >= platforms.value.length) {
    gameOver()
    return
  }
  
  const nextPlatform = platforms.value[nextPlatformIndex]
  const currentPlatform = platforms.value[p.currentPlatform]
  
  if (p.vy > 0) {
    const playerBottom = p.y
    const nextPlatformTop = nextPlatform.y
    const currentPlatformTop = currentPlatform.y
    
    if (playerBottom >= nextPlatformTop && playerBottom <= nextPlatformTop + 30) {
      const playerLeft = p.x - PLAYER_SIZE / 2
      const playerRight = p.x + PLAYER_SIZE / 2
      
      if (playerRight > nextPlatform.x && playerLeft < nextPlatform.x + nextPlatform.width) {
        p.y = nextPlatformTop
        p.vy = 0
        p.vx = 0
        p.isJumping = false
        p.currentPlatform = nextPlatformIndex
        
        score.value++
        if (score.value > bestScore.value) {
          bestScore.value = score.value
        }
        
        generatePlatform(platforms.value.length)
        
        adjustView()
        return
      }
    }
    
    if (playerBottom > currentPlatformTop + 100) {
      const playerLeft = p.x - PLAYER_SIZE / 2
      const playerRight = p.x + PLAYER_SIZE / 2
      
      if (playerRight > currentPlatform.x && playerLeft < currentPlatform.x + currentPlatform.width) {
        p.y = currentPlatformTop
        p.vy = 0
        p.vx = 0
        p.isJumping = false
        return
      }
    }
    
    if (playerBottom > canvasHeight + 100) {
      gameOver()
    }
  }
}

function adjustView() {
  const currentPlatform = platforms.value[player.value.currentPlatform]
  const targetY = canvasHeight - 200
  const offset = targetY - currentPlatform.y
  
  if (offset > 0) {
    platforms.value.forEach(platform => {
      platform.y += offset
    })
    player.value.y += offset
  }
}

function gameOver() {
  player.value.isJumping = false
  gameState.value = 'GAME_OVER'
  
  if (animationId) {
    cancelAnimationFrame(animationId)
    animationId = null
  }
  
  if (chargeInterval) {
    clearInterval(chargeInterval)
    chargeInterval = null
  }
}

onMounted(() => {
  if (canvasRef.value) {
    ctx = canvasRef.value.getContext('2d')
    initGame()
    startRender()
  }
})

onUnmounted(() => {
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
  if (chargeInterval) {
    clearInterval(chargeInterval)
  }
})
</script>

<style scoped>
.jump-game {
  position: relative;
  width: 100%;
  max-width: 500px;
  height: 100vh;
  max-height: 700px;
  display: flex;
  flex-direction: column;
  align-items: center;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-radius: 12px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  overflow: hidden;
}

.game-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
  padding: 16px 20px;
  background: rgba(0, 0, 0, 0.2);
  backdrop-filter: blur(10px);
  z-index: 10;
}

.score, .best-score {
  display: flex;
  flex-direction: column;
  align-items: center;
  color: #fff;
}

.label {
  font-size: 12px;
  opacity: 0.8;
  margin-bottom: 4px;
}

.value {
  font-size: 28px;
  font-weight: bold;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
}

.game-canvas {
  flex: 1;
  width: 100%;
  height: calc(100% - 60px);
  cursor: pointer;
  touch-action: none;
}

.game-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 20;
  backdrop-filter: blur(5px);
}

.overlay-content {
  text-align: center;
  color: #fff;
  padding: 40px;
}

.overlay-content h1 {
  font-size: 48px;
  margin-bottom: 16px;
  text-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
}

.final-score {
  font-size: 24px;
  margin-bottom: 12px;
  opacity: 0.9;
}

.hint {
  font-size: 16px;
  margin-bottom: 24px;
  opacity: 0.7;
}

.start-btn, .restart-btn {
  padding: 16px 48px;
  font-size: 18px;
  font-weight: bold;
  color: #fff;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border: none;
  border-radius: 30px;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 4px 15px rgba(102, 126, 234, 0.4);
}

.start-btn:hover, .restart-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(102, 126, 234, 0.6);
}

.start-btn:active, .restart-btn:active {
  transform: translateY(0);
}

.charge-indicator {
  position: absolute;
  bottom: 80px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  flex-direction: column;
  align-items: center;
  z-index: 15;
}

.charge-bar {
  width: 200px;
  height: 20px;
  background: rgba(0, 0, 0, 0.5);
  border-radius: 10px;
  overflow: hidden;
  border: 2px solid rgba(255, 255, 255, 0.3);
}

.charge-fill {
  height: 100%;
  background: linear-gradient(90deg, #4ECDC4, #45B7D1);
  transition: width 0.05s linear;
  border-radius: 8px;
}

.charge-label {
  margin-top: 8px;
  color: #fff;
  font-size: 14px;
  opacity: 0.8;
}

.tips {
  position: absolute;
  bottom: 20px;
  left: 0;
  right: 0;
  text-align: center;
  z-index: 10;
}

.tips p {
  color: rgba(255, 255, 255, 0.6);
  font-size: 14px;
}
</style>
