<template>
  <div class="game-wrapper">
    <div class="jump-game" :style="gameContainerStyle">
      <div class="game-header" v-if="gameState !== 'IDLE'">
        <div class="score-display">
          <div class="score-item current-score">
            <span class="score-label">分数</span>
            <span class="score-value" :class="{ 'score-animate': scoreChanged }">{{ displayScore }}</span>
          </div>
          <div class="score-divider"></div>
          <div class="score-item best-score">
            <span class="score-label">最高分</span>
            <span class="score-value">{{ bestScore }}</span>
          </div>
        </div>
        <div class="combo-display" v-if="combo > 1">
          <span class="combo-text">
            <span class="combo-icon">✨</span>
            完美 x{{ combo }}
            <span class="combo-bonus" v-if="combo >= 3">+{{ Math.floor(combo / 3) }}</span>
          </span>
        </div>
        <div class="difficulty-indicator" v-if="difficultyLevel > 1 && gameState === 'PLAYING'">
          <span class="difficulty-text">难度 Lv.{{ difficultyLevel }}</span>
        </div>
      </div>

      <div class="canvas-container">
        <canvas
          ref="canvasRef"
          :width="canvasDesignWidth"
          :height="canvasDesignHeight"
          class="game-canvas"
          :style="canvasStyle"
          @mousedown="handleMouseDown"
          @mouseup="handleMouseUp"
          @mouseleave="handleMouseUp"
          @touchstart.prevent="handleMouseDown"
          @touchend.prevent="handleMouseUp"
        ></canvas>

        <div class="charge-overlay" v-if="gameState === 'CHARGING'">
          <div class="circular-charge">
            <svg class="charge-svg" viewBox="0 0 100 100">
              <circle class="charge-ring-bg" cx="50" cy="50" r="42"></circle>
              <circle 
                class="charge-ring" 
                cx="50" 
                cy="50" 
                r="42"
                :style="{ strokeDashoffset: 264 - (264 * chargePercent / 100) }"
              ></circle>
            </svg>
            <div class="charge-percentage">{{ Math.round(chargePercent) }}</div>
          </div>
          <div class="charge-hint">松开跳跃！</div>
        </div>

        <div class="direction-indicator" v-if="gameState === 'CHARGING' && nextPlatform">
          <div class="arrow" :class="{ 'arrow-left': direction < 0, 'arrow-right': direction > 0 }">
            <span class="arrow-icon">{{ direction > 0 ? '→' : '←' }}</span>
            <span class="direction-text">{{ direction > 0 ? '向右' : '向左' }}</span>
          </div>
        </div>
      </div>

      <div class="game-footer" v-if="gameState === 'PLAYING' && !player.isJumping">
        <div class="hint-text">
          <span class="hint-icon">👆</span>
          <span>长按蓄力 · 松开跳跃</span>
        </div>
      </div>

      <transition name="fade">
        <div class="game-overlay" v-if="gameState === 'IDLE'">
          <div class="overlay-content">
            <div class="game-logo">
              <div class="logo-icon">🎯</div>
            </div>
            <h1 class="game-title">跳一跳</h1>
            <p class="game-desc">经典小游戏 · 挑战最高分</p>
            
            <div class="game-tutorial" v-if="!hasPlayed">
              <div class="tutorial-step">
                <div class="step-num">1</div>
                <div class="step-content">
                  <span class="step-title">长按蓄力</span>
                  <span class="step-desc">按住屏幕蓄力，蓄力越久跳得越远</span>
                </div>
              </div>
              <div class="tutorial-step">
                <div class="step-num">2</div>
                <div class="step-content">
                  <span class="step-title">观察方向</span>
                  <span class="step-desc">注意下一个平台的方向（左/右）</span>
                </div>
              </div>
              <div class="tutorial-step">
                <div class="step-num">3</div>
                <div class="step-content">
                  <span class="step-title">精准跳跃</span>
                  <span class="step-desc">跳到平台中心获得「完美」连击加成</span>
                </div>
              </div>
            </div>

            <div class="quick-stats" v-if="hasPlayed && bestScore > 0">
              <span class="stat-label">历史最高</span>
              <span class="stat-value">{{ bestScore }}</span>
            </div>

            <button class="start-btn" @click="startGame">
              <span class="btn-text">开始游戏</span>
              <span class="btn-icon">▶</span>
            </button>
          </div>
        </div>
      </transition>

      <transition name="fade">
        <div class="game-overlay" v-if="gameState === 'GAME_OVER'">
          <div class="overlay-content">
            <div class="result-icon" :class="{ 'new-record': isNewRecord }">
              <span v-if="isNewRecord">🏆</span>
              <span v-else>💫</span>
            </div>
            <h1 class="result-title">
              {{ isNewRecord ? '新纪录！' : '游戏结束' }}
            </h1>
            
            <div class="final-score">
              <span class="final-label">最终分数</span>
              <span class="final-value">{{ score }}</span>
            </div>
            
            <div class="result-stats" v-if="combo > 0 || bestScore > 0">
              <div class="stat-row" v-if="maxCombo > 1">
                <span class="stat-desc">最高连击</span>
                <span class="stat-num">x{{ maxCombo }}</span>
              </div>
              <div class="stat-row">
                <span class="stat-desc">历史最高</span>
                <span class="stat-num">{{ bestScore }}</span>
              </div>
            </div>

            <button class="restart-btn" @click="startGame">
              <span class="btn-icon">🔄</span>
              <span class="btn-text">再来一局</span>
            </button>
          </div>
        </div>
      </transition>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed, watch, nextTick } from 'vue'

const CANVAS_RATIO = 3 / 4
const canvasRef = ref(null)
const canvasDesignWidth = 480
const canvasDesignHeight = 640

const containerWidth = ref(0)
const containerHeight = ref(0)

const gameContainerStyle = computed(() => ({
  width: `${Math.min(520, containerWidth.value)}px`,
  height: `${Math.min(800, containerHeight.value)}px`
}))

const canvasScale = computed(() => {
  if (!canvasRef.value) return 1
  const container = canvasRef.value.parentElement
  if (!container) return 1
  
  const availableWidth = container.clientWidth - 40
  const availableHeight = container.clientHeight - 20
  
  const scaleX = availableWidth / canvasDesignWidth
  const scaleY = availableHeight / canvasDesignHeight
  
  return Math.min(scaleX, scaleY, 1.2)
})

const canvasStyle = computed(() => ({
  transform: `scale(${canvasScale.value})`,
  transformOrigin: 'center center'
}))

const GAME_CONSTANTS = {
  GRAVITY: 0.4,
  MAX_CHARGE: 100,
  CHARGE_SPEED: 1.2,
  PLATFORM_HEIGHT: 25,
  PLAYER_WIDTH: 24,
  PLAYER_HEIGHT: 40,
  BASE_PLATFORM_MIN_WIDTH: 60,
  BASE_PLATFORM_MAX_WIDTH: 100,
  BASE_PLATFORM_MIN_GAP: 80,
  BASE_PLATFORM_MAX_GAP: 140,
  DIFFICULTY_INCREASE_SCORE: 5,
  PERFECT_THRESHOLD: 10,
  STORAGE_KEY: 'jump_game_best_score'
}

const gameState = ref('IDLE')
const score = ref(0)
const displayScore = ref(0)
const bestScore = ref(0)
const combo = ref(0)
const maxCombo = ref(0)
const scoreChanged = ref(false)
const chargeTime = ref(0)
const chargePercent = computed(() => Math.min(100, (chargeTime.value / GAME_CONSTANTS.MAX_CHARGE) * 100))
const hasPlayed = ref(false)
const isNewRecord = ref(false)

const difficultyLevel = computed(() => {
  return Math.floor(score.value / GAME_CONSTANTS.DIFFICULTY_INCREASE_SCORE) + 1
})

const currentDifficulty = computed(() => {
  const level = difficultyLevel.value
  const difficultyMultiplier = Math.min(1 + (level - 1) * 0.15, 2.5)
  
  return {
    minGap: GAME_CONSTANTS.BASE_PLATFORM_MIN_GAP + (level - 1) * 8,
    maxGap: Math.min(GAME_CONSTANTS.BASE_PLATFORM_MAX_GAP + (level - 1) * 12, 220),
    minWidth: Math.max(GAME_CONSTANTS.BASE_PLATFORM_MIN_WIDTH - (level - 1) * 4, 40),
    maxWidth: Math.max(GAME_CONSTANTS.BASE_PLATFORM_MAX_WIDTH - (level - 1) * 6, 50)
  }
})

let ctx = null
let animationId = null
let lastTime = 0
let isGameRunning = false
let scoreAnimationId = null

const platforms = ref([])
const player = ref({
  x: 0,
  y: 0,
  vx: 0,
  vy: 0,
  isJumping: false,
  currentPlatform: 0,
  squash: 1,
  stretch: 1
})

const nextPlatform = computed(() => {
  const nextIndex = player.value.currentPlatform + 1
  if (nextIndex < platforms.value.length) {
    return platforms.value[nextIndex]
  }
  return null
})

const direction = computed(() => {
  if (!nextPlatform.value) return 0
  const currentPlatform = platforms.value[player.value.currentPlatform]
  if (!currentPlatform) return 0
  return nextPlatform.value.x + nextPlatform.value.width / 2 > currentPlatform.x + currentPlatform.width / 2 ? 1 : -1
})

const particles = ref([])
const landingEffect = ref({
  active: false,
  x: 0,
  y: 0,
  radius: 0,
  opacity: 1
})

const platformColors = [
  { main: '#FF6B9D', light: '#FF8FB1', dark: '#D94F7D' },
  { main: '#9B59B6', light: '#B078C9', dark: '#7D3C98' },
  { main: '#3498DB', light: '#5DADE2', dark: '#2980B9' },
  { main: '#1ABC9C', light: '#48C9B0', dark: '#16A085' },
  { main: '#F39C12', light: '#F5B041', dark: '#D68910' },
  { main: '#E74C3C', light: '#EC7063', dark: '#C0392B' },
  { main: '#00D2D3', light: '#4CE0E0', dark: '#00A8A8' },
  { main: '#A8E6CF', light: '#C7F0DB', dark: '#7ED6B3' }
]

const backgroundStars = []
for (let i = 0; i < 50; i++) {
  backgroundStars.push({
    x: Math.random() * canvasDesignWidth,
    y: Math.random() * canvasDesignHeight,
    size: Math.random() * 2 + 0.5,
    opacity: Math.random() * 0.5 + 0.2,
    twinkleSpeed: Math.random() * 0.02 + 0.01
  })
}

function loadBestScore() {
  try {
    const saved = localStorage.getItem(GAME_CONSTANTS.STORAGE_KEY)
    if (saved) {
      bestScore.value = parseInt(saved, 10) || 0
      hasPlayed.value = bestScore.value > 0
    }
  } catch (e) {
    console.warn('无法读取 localStorage:', e)
  }
}

function saveBestScore() {
  try {
    localStorage.setItem(GAME_CONSTANTS.STORAGE_KEY, bestScore.value.toString())
  } catch (e) {
    console.warn('无法写入 localStorage:', e)
  }
}

function updateDisplayScore(target) {
  if (scoreAnimationId) {
    cancelAnimationFrame(scoreAnimationId)
    scoreAnimationId = null
  }
  
  const start = displayScore.value
  const diff = target - start
  if (diff === 0) return
  
  const duration = 300
  const startTime = performance.now()
  
  function animate(currentTime) {
    const elapsed = currentTime - startTime
    const progress = Math.min(elapsed / duration, 1)
    
    const easeOut = 1 - Math.pow(1 - progress, 3)
    displayScore.value = Math.round(start + diff * easeOut)
    
    if (progress < 1) {
      scoreAnimationId = requestAnimationFrame(animate)
    }
  }
  
  scoreAnimationId = requestAnimationFrame(animate)
}

function initGame() {
  platforms.value = []
  
  const firstWidth = 110
  const firstPlatform = {
    x: canvasDesignWidth / 2 - firstWidth / 2,
    y: canvasDesignHeight - 180,
    width: firstWidth,
    height: GAME_CONSTANTS.PLATFORM_HEIGHT,
    color: platformColors[0],
    radius: 8
  }
  platforms.value.push(firstPlatform)
  
  for (let i = 1; i < 15; i++) {
    generatePlatform(i)
  }
  
  player.value = {
    x: canvasDesignWidth / 2,
    y: firstPlatform.y,
    vx: 0,
    vy: 0,
    isJumping: false,
    currentPlatform: 0,
    squash: 1,
    stretch: 1
  }
  
  score.value = 0
  displayScore.value = 0
  combo.value = 0
  maxCombo.value = 0
  chargeTime.value = 0
  particles.value = []
  landingEffect.value = { active: false, x: 0, y: 0, radius: 0, opacity: 1 }
  isNewRecord.value = false
}

function generatePlatform(index) {
  const prevPlatform = platforms.value[index - 1]
  const diff = currentDifficulty.value
  
  const gap = diff.minGap + Math.random() * (diff.maxGap - diff.minGap)
  const width = diff.minWidth + Math.random() * (diff.maxWidth - diff.minWidth)
  
  const dir = Math.random() > 0.5 ? 1 : -1
  const xOffset = (Math.random() * 60 + 40) * dir
  
  let newX = prevPlatform.x + prevPlatform.width / 2 + xOffset - width / 2
  newX = Math.max(30, Math.min(canvasDesignWidth - width - 30, newX))
  
  platforms.value.push({
    x: newX,
    y: prevPlatform.y - gap,
    width: width,
    height: GAME_CONSTANTS.PLATFORM_HEIGHT,
    color: platformColors[index % platformColors.length],
    radius: 6 + Math.random() * 4
  })
}

function startGame() {
  if (scoreAnimationId) {
    cancelAnimationFrame(scoreAnimationId)
    scoreAnimationId = null
  }
  
  const previousBest = bestScore.value
  initGame()
  hasPlayed.value = true
  bestScore.value = previousBest
  
  gameState.value = 'PLAYING'
  lastTime = performance.now()
  isGameRunning = true
  
  stopRenderLoop()
  startRenderLoop()
}

function startRenderLoop() {
  if (!isGameRunning) return
  gameLoop()
}

function stopRenderLoop() {
  isGameRunning = false
  if (animationId) {
    cancelAnimationFrame(animationId)
    animationId = null
  }
}

function gameLoop(currentTime = performance.now()) {
  if (!isGameRunning) return
  
  const deltaTime = Math.min((currentTime - lastTime) / 16.67, 2)
  lastTime = currentTime
  
  update(deltaTime)
  render()
  
  animationId = requestAnimationFrame(gameLoop)
}

function update(deltaTime) {
  if (player.value.isJumping) {
    updatePhysics(deltaTime)
  }
  
  if (gameState.value === 'CHARGING') {
    player.value.squash = Math.max(0.6, 1 - chargePercent.value / 300)
    player.value.stretch = Math.min(1.3, 1 + chargePercent.value / 500)
  } else if (!player.value.isJumping) {
    player.value.squash += (1 - player.value.squash) * 0.1
    player.value.stretch += (1 - player.value.stretch) * 0.1
  }
  
  particles.value = particles.value.filter(p => {
    p.x += p.vx * deltaTime
    p.y += p.vy * deltaTime
    p.vy += 0.1 * deltaTime
    p.life -= 0.02 * deltaTime
    p.opacity = Math.max(0, p.life)
    return p.life > 0
  })
  
  if (landingEffect.value.active) {
    landingEffect.value.radius += 3 * deltaTime
    landingEffect.value.opacity -= 0.05 * deltaTime
    if (landingEffect.value.opacity <= 0) {
      landingEffect.value.active = false
    }
  }
  
  backgroundStars.forEach(star => {
    star.opacity = 0.2 + Math.abs(Math.sin(performance.now() * star.twinkleSpeed)) * 0.5
  })
}

function updatePhysics(deltaTime) {
  player.value.x += player.value.vx * deltaTime
  player.value.y += player.value.vy * deltaTime
  player.value.vy += GAME_CONSTANTS.GRAVITY * deltaTime
  
  if (player.value.vy > 0) {
    player.value.squash = Math.min(1.2, 1 + player.value.vy / 50)
    player.value.stretch = Math.max(0.8, 1 - player.value.vy / 100)
  }
  
  checkLanding()
}

function checkLanding() {
  const p = player.value
  const nextPlatformIndex = p.currentPlatform + 1
  
  const platformsToCheck = []
  if (nextPlatformIndex < platforms.value.length) {
    platformsToCheck.push({ index: nextPlatformIndex, platform: platforms.value[nextPlatformIndex] })
  }
  if (p.currentPlatform < platforms.value.length) {
    platformsToCheck.push({ index: p.currentPlatform, platform: platforms.value[p.currentPlatform] })
  }
  
  for (const { index, platform } of platformsToCheck) {
    const playerBottom = p.y
    const platformTop = platform.y
    const platformBottom = platform.y + platform.height
    
    if (playerBottom >= platformTop - 5 && playerBottom <= platformBottom + 15) {
      const playerLeft = p.x - GAME_CONSTANTS.PLAYER_WIDTH / 2
      const playerRight = p.x + GAME_CONSTANTS.PLAYER_WIDTH / 2
      const platformLeft = platform.x
      const platformRight = platform.x + platform.width
      
      const overlapLeft = Math.max(playerLeft, platformLeft)
      const overlapRight = Math.min(playerRight, platformRight)
      
      if (overlapRight > overlapLeft) {
        p.y = platformTop
        p.vy = 0
        p.vx = 0
        p.isJumping = false
        
        p.squash = 0.65
        p.stretch = 1.25
        
        createLandingEffect(p.x, platformTop)
        
        if (index !== p.currentPlatform) {
          p.currentPlatform = index
          
          const platformCenterX = platform.x + platform.width / 2
          const distanceFromCenter = Math.abs(p.x - platformCenterX)
          
          if (distanceFromCenter < GAME_CONSTANTS.PERFECT_THRESHOLD) {
            combo.value++
            if (combo.value > maxCombo.value) {
              maxCombo.value = combo.value
            }
            const bonusScore = 1 + Math.floor(combo.value / 3)
            score.value += bonusScore
          } else {
            combo.value = 1
            score.value += 1
          }
          
          scoreChanged.value = true
          updateDisplayScore(score.value)
          setTimeout(() => { scoreChanged.value = false }, 300)
          
          if (score.value > bestScore.value) {
            bestScore.value = score.value
            saveBestScore()
          }
          
          generatePlatform(platforms.value.length)
          smoothAdjustView()
        }
        
        return
      }
    }
  }
  
  if (p.y > canvasDesignHeight + 250) {
    gameOver()
  }
}

function createLandingEffect(x, y) {
  landingEffect.value = {
    active: true,
    x: x,
    y: y,
    radius: 5,
    opacity: 1
  }
  
  const particleCount = combo.value > 1 ? 12 : 8
  for (let i = 0; i < particleCount; i++) {
    const angle = (Math.PI * 2 * i) / particleCount + Math.random() * 0.4
    const speed = 2 + Math.random() * 3 + (combo.value > 1 ? 1 : 0)
    particles.value.push({
      x: x,
      y: y,
      vx: Math.cos(angle) * speed,
      vy: -Math.random() * 3 - 1.5,
      size: 3 + Math.random() * 3,
      color: `hsl(${Math.random() * 60 + 0}, 100%, 70%)`,
      life: 1,
      opacity: 1
    })
  }
}

function smoothAdjustView() {
  const currentPlatform = platforms.value[player.value.currentPlatform]
  const targetY = canvasDesignHeight - 220
  const offset = targetY - currentPlatform.y
  
  if (offset > 0) {
    platforms.value.forEach(platform => {
      platform.y += offset
    })
    player.value.y += offset
  }
}

function render() {
  if (!ctx) return
  
  drawBackground()
  drawPlatforms()
  drawLandingEffect()
  drawParticles()
  drawPlayer()
}

function drawBackground() {
  const gradient = ctx.createLinearGradient(0, 0, 0, canvasDesignHeight)
  gradient.addColorStop(0, '#1a1a2e')
  gradient.addColorStop(0.5, '#16213e')
  gradient.addColorStop(1, '#0f0f23')
  ctx.fillStyle = gradient
  ctx.fillRect(0, 0, canvasDesignWidth, canvasDesignHeight)
  
  backgroundStars.forEach(star => {
    ctx.fillStyle = `rgba(255, 255, 255, ${star.opacity})`
    ctx.beginPath()
    ctx.arc(star.x, star.y, star.size, 0, Math.PI * 2)
    ctx.fill()
  })
  
  ctx.strokeStyle = 'rgba(255, 255, 255, 0.02)'
  ctx.lineWidth = 1
  
  for (let x = 0; x < canvasDesignWidth; x += 40) {
    ctx.beginPath()
    ctx.moveTo(x, 0)
    ctx.lineTo(x, canvasDesignHeight)
    ctx.stroke()
  }
  
  for (let y = 0; y < canvasDesignHeight; y += 40) {
    ctx.beginPath()
    ctx.moveTo(0, y)
    ctx.lineTo(canvasDesignWidth, y)
    ctx.stroke()
  }
}

function drawPlatforms() {
  platforms.value.forEach((platform, index) => {
    const isCurrentPlatform = index === player.value.currentPlatform && !player.value.isJumping
    const isNextPlatform = index === player.value.currentPlatform + 1
    const { x, y, width, height, color, radius } = platform
    
    ctx.fillStyle = 'rgba(0, 0, 0, 0.35)'
    roundRect(ctx, x + 5, y + 5, width, height, radius)
    ctx.fill()
    
    const bodyGradient = ctx.createLinearGradient(x, y, x, y + height)
    bodyGradient.addColorStop(0, color.light)
    bodyGradient.addColorStop(0.5, color.main)
    bodyGradient.addColorStop(1, color.dark)
    ctx.fillStyle = bodyGradient
    roundRect(ctx, x, y, width, height, radius)
    ctx.fill()
    
    ctx.fillStyle = 'rgba(255, 255, 255, 0.35)'
    roundRect(ctx, x + 3, y + 3, width - 6, height / 3, radius - 2)
    ctx.fill()
    
    if (isCurrentPlatform) {
      ctx.strokeStyle = 'rgba(255, 255, 255, 0.5)'
      ctx.lineWidth = 2
      roundRect(ctx, x - 1, y - 1, width + 2, height + 2, radius + 1)
      ctx.stroke()
      
      const centerX = x + width / 2
      ctx.fillStyle = 'rgba(255, 255, 255, 0.9)'
      ctx.beginPath()
      ctx.arc(centerX, y - 8, 3, 0, Math.PI * 2)
      ctx.fill()
    }
    
    if (isNextPlatform && gameState.value === 'PLAYING' && !player.value.isJumping) {
      const centerX = x + width / 2
      const blink = 0.3 + Math.abs(Math.sin(performance.now() * 0.003)) * 0.4
      
      ctx.fillStyle = `rgba(255, 255, 255, ${blink})`
      ctx.beginPath()
      ctx.arc(centerX, y - 12, 4, 0, Math.PI * 2)
      ctx.fill()
    }
  })
}

function drawPlayer() {
  const p = player.value
  
  ctx.save()
  ctx.translate(p.x, p.y)
  ctx.scale(1 / p.squash, 1 / p.stretch)
  
  ctx.fillStyle = 'rgba(0, 0, 0, 0.3)'
  ctx.beginPath()
  ctx.ellipse(3, GAME_CONSTANTS.PLAYER_HEIGHT / 2 + 5, GAME_CONSTANTS.PLAYER_WIDTH / 2, 6, 0, 0, Math.PI * 2)
  ctx.fill()
  
  const bodyGradient = ctx.createLinearGradient(
    -GAME_CONSTANTS.PLAYER_WIDTH / 2, -GAME_CONSTANTS.PLAYER_HEIGHT,
    GAME_CONSTANTS.PLAYER_WIDTH / 2, 0
  )
  bodyGradient.addColorStop(0, '#FF6B6B')
  bodyGradient.addColorStop(0.5, '#EE5A5A')
  bodyGradient.addColorStop(1, '#C0392B')
  ctx.fillStyle = bodyGradient
  
  ctx.beginPath()
  ctx.moveTo(-GAME_CONSTANTS.PLAYER_WIDTH / 2 + 4, 0)
  ctx.quadraticCurveTo(-GAME_CONSTANTS.PLAYER_WIDTH / 2, 0, -GAME_CONSTANTS.PLAYER_WIDTH / 2, -4)
  ctx.lineTo(-GAME_CONSTANTS.PLAYER_WIDTH / 2, -GAME_CONSTANTS.PLAYER_HEIGHT + 6)
  ctx.quadraticCurveTo(-GAME_CONSTANTS.PLAYER_WIDTH / 2, -GAME_CONSTANTS.PLAYER_HEIGHT, -GAME_CONSTANTS.PLAYER_WIDTH / 2 + 6, -GAME_CONSTANTS.PLAYER_HEIGHT)
  ctx.lineTo(GAME_CONSTANTS.PLAYER_WIDTH / 2 - 6, -GAME_CONSTANTS.PLAYER_HEIGHT)
  ctx.quadraticCurveTo(GAME_CONSTANTS.PLAYER_WIDTH / 2, -GAME_CONSTANTS.PLAYER_HEIGHT, GAME_CONSTANTS.PLAYER_WIDTH / 2, -GAME_CONSTANTS.PLAYER_HEIGHT + 6)
  ctx.lineTo(GAME_CONSTANTS.PLAYER_WIDTH / 2, -4)
  ctx.quadraticCurveTo(GAME_CONSTANTS.PLAYER_WIDTH / 2, 0, GAME_CONSTANTS.PLAYER_WIDTH / 2 - 4, 0)
  ctx.closePath()
  ctx.fill()
  
  ctx.fillStyle = 'rgba(255, 255, 255, 0.3)'
  ctx.beginPath()
  ctx.ellipse(-GAME_CONSTANTS.PLAYER_WIDTH / 4, -GAME_CONSTANTS.PLAYER_HEIGHT / 2, 3, GAME_CONSTANTS.PLAYER_HEIGHT / 3, 0, 0, Math.PI * 2)
  ctx.fill()
  
  const eyeOffsetX = 5
  const eyeOffsetY = -GAME_CONSTANTS.PLAYER_HEIGHT / 2 - 4
  const eyeSize = 4
  
  ctx.fillStyle = '#FFF'
  ctx.beginPath()
  ctx.arc(-eyeOffsetX, eyeOffsetY, eyeSize, 0, Math.PI * 2)
  ctx.arc(eyeOffsetX, eyeOffsetY, eyeSize, 0, Math.PI * 2)
  ctx.fill()
  
  const pupilOffset = direction.value * 1.5
  ctx.fillStyle = '#2C3E50'
  ctx.beginPath()
  ctx.arc(-eyeOffsetX + pupilOffset, eyeOffsetY, 2, 0, Math.PI * 2)
  ctx.arc(eyeOffsetX + pupilOffset, eyeOffsetY, 2, 0, Math.PI * 2)
  ctx.fill()
  
  ctx.fillStyle = 'rgba(255, 150, 150, 0.5)'
  ctx.beginPath()
  ctx.ellipse(-eyeOffsetX - 5, eyeOffsetY + 5, 3, 2, 0, 0, Math.PI * 2)
  ctx.ellipse(eyeOffsetX + 5, eyeOffsetY + 5, 3, 2, 0, 0, Math.PI * 2)
  ctx.fill()
  
  if (gameState.value === 'CHARGING') {
    const pulseSize = 18 + chargePercent.value / 6
    const pulseOpacity = 0.15 + chargePercent.value / 200
    
    ctx.fillStyle = `rgba(255, 255, 255, ${pulseOpacity})`
    ctx.beginPath()
    ctx.arc(0, -GAME_CONSTANTS.PLAYER_HEIGHT / 2, pulseSize, 0, Math.PI * 2)
    ctx.fill()
    
    for (let i = 0; i < 5; i++) {
      const particleAngle = (Math.PI / 2) + (i - 2) * 0.4
      const particleDist = pulseSize + 6 + Math.sin(performance.now() * 0.01 + i) * 4
      const px = Math.cos(particleAngle) * particleDist
      const py = Math.sin(particleAngle) * particleDist - GAME_CONSTANTS.PLAYER_HEIGHT / 2
      
      ctx.fillStyle = `rgba(255, 255, 255, ${0.4 + chargePercent.value / 180})`
      ctx.beginPath()
      ctx.arc(px, py, 2.5 + chargePercent.value / 45, 0, Math.PI * 2)
      ctx.fill()
    }
  }
  
  ctx.restore()
}

function drawLandingEffect() {
  if (!landingEffect.value.active) return
  
  const { x, y, radius, opacity } = landingEffect.value
  
  ctx.strokeStyle = `rgba(255, 255, 255, ${opacity})`
  ctx.lineWidth = 2
  ctx.beginPath()
  ctx.arc(x, y, radius, 0, Math.PI * 2)
  ctx.stroke()
  
  ctx.strokeStyle = `rgba(255, 200, 100, ${opacity * 0.6})`
  ctx.lineWidth = 3
  ctx.beginPath()
  ctx.arc(x, y, radius * 0.6, 0, Math.PI * 2)
  ctx.stroke()
  
  if (combo.value > 1) {
    ctx.strokeStyle = `rgba(255, 100, 150, ${opacity * 0.4})`
    ctx.lineWidth = 4
    ctx.beginPath()
    ctx.arc(x, y, radius * 0.3, 0, Math.PI * 2)
    ctx.stroke()
  }
}

function drawParticles() {
  particles.value.forEach(p => {
    ctx.fillStyle = p.color.replace(')', `, ${p.opacity})`).replace('hsl', 'hsla')
    ctx.beginPath()
    ctx.arc(p.x, p.y, p.size * p.life, 0, Math.PI * 2)
    ctx.fill()
  })
}

function roundRect(ctx, x, y, width, height, radius) {
  const r = Math.min(radius, width / 2, height / 2)
  ctx.beginPath()
  ctx.moveTo(x + r, y)
  ctx.lineTo(x + width - r, y)
  ctx.quadraticCurveTo(x + width, y, x + width, y + r)
  ctx.lineTo(x + width, y + height - r)
  ctx.quadraticCurveTo(x + width, y + height, x + width - r, y + height)
  ctx.lineTo(x + r, y + height)
  ctx.quadraticCurveTo(x, y + height, x, y + height - r)
  ctx.lineTo(x, y + r)
  ctx.quadraticCurveTo(x, y, x + r, y)
  ctx.closePath()
}

function handleMouseDown() {
  if (gameState.value !== 'PLAYING' || player.value.isJumping) return
  
  gameState.value = 'CHARGING'
  chargeTime.value = 0
  
  startChargeAnimation()
}

function startChargeAnimation() {
  const animate = () => {
    if (gameState.value === 'CHARGING') {
      if (chargeTime.value < GAME_CONSTANTS.MAX_CHARGE) {
        chargeTime.value += GAME_CONSTANTS.CHARGE_SPEED
      }
      requestAnimationFrame(animate)
    }
  }
  requestAnimationFrame(animate)
}

function handleMouseUp() {
  if (gameState.value !== 'CHARGING') return
  
  const currentPlatform = platforms.value[player.value.currentPlatform]
  const nextPlat = platforms.value[player.value.currentPlatform + 1]
  
  if (!nextPlat) {
    gameOver()
    return
  }
  
  const dir = nextPlat.x + nextPlat.width / 2 > currentPlatform.x + currentPlatform.width / 2 ? 1 : -1
  const power = chargePercent.value / 100
  
  const jumpVx = (power * 6.5 + 1.5) * dir
  const jumpVy = -(power * 10 + 5)
  
  player.value.vx = jumpVx
  player.value.vy = jumpVy
  player.value.isJumping = true
  player.value.squash = 1.15
  player.value.stretch = 0.85
  
  gameState.value = 'PLAYING'
  chargeTime.value = 0
}

function gameOver() {
  player.value.isJumping = false
  
  if (score.value > 0 && score.value >= bestScore.value) {
    isNewRecord.value = true
  }
  
  gameState.value = 'GAME_OVER'
  stopRenderLoop()
  
  updateDisplayScore(score.value)
}

function updateContainerSize() {
  containerWidth.value = window.innerWidth
  containerHeight.value = window.innerHeight
}

function handleResize() {
  updateContainerSize()
}

onMounted(() => {
  loadBestScore()
  updateContainerSize()
  
  if (canvasRef.value) {
    ctx = canvasRef.value.getContext('2d')
    ctx.imageSmoothingEnabled = true
    ctx.imageSmoothingQuality = 'high'
    
    initGame()
    gameState.value = 'IDLE'
    render()
  }
  
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  stopRenderLoop()
  
  if (scoreAnimationId) {
    cancelAnimationFrame(scoreAnimationId)
    scoreAnimationId = null
  }
  
  window.removeEventListener('resize', handleResize)
})
</script>

<style scoped>
.game-wrapper {
  width: 100vw;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f0f23 100%);
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  overflow: hidden;
}

.jump-game {
  position: relative;
  display: flex;
  flex-direction: column;
  background: rgba(20, 20, 40, 0.98);
  border-radius: 20px;
  box-shadow: 
    0 25px 80px rgba(0, 0, 0, 0.6),
    0 0 0 1px rgba(255, 255, 255, 0.06),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
  overflow: hidden;
  margin: 16px;
}

.game-header {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 14px 20px 8px;
  background: linear-gradient(
    to bottom,
    rgba(0, 0, 0, 0.45),
    rgba(0, 0, 0, 0.25),
    transparent
  );
  z-index: 10;
  flex-shrink: 0;
}

.score-display {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 0;
  width: 100%;
}

.score-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  min-width: 80px;
}

.score-divider {
  width: 1px;
  height: 30px;
  background: rgba(255, 255, 255, 0.1);
  margin: 0 20px;
}

.score-label {
  font-size: 10px;
  color: rgba(255, 255, 255, 0.4);
  text-transform: uppercase;
  letter-spacing: 1.5px;
  margin-bottom: 3px;
}

.score-value {
  font-size: 30px;
  font-weight: 700;
  color: #fff;
  text-shadow: 0 2px 12px rgba(255, 255, 255, 0.15);
  line-height: 1;
  font-variant-numeric: tabular-nums;
}

.score-animate {
  animation: scorePop 0.35s cubic-bezier(0.34, 1.56, 0.64, 1);
}

@keyframes scorePop {
  0% { transform: scale(1); color: #fff; }
  50% { transform: scale(1.35); color: #FFD700; }
  100% { transform: scale(1); color: #fff; }
}

.combo-display {
  margin-top: 6px;
}

.combo-text {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 13px;
  font-weight: 600;
  color: #FFD700;
  text-shadow: 0 0 15px rgba(255, 215, 0, 0.5);
  animation: comboPulse 0.5s ease-in-out infinite;
}

.combo-icon {
  font-size: 14px;
}

.combo-bonus {
  font-size: 11px;
  color: #FFA500;
  background: rgba(255, 165, 0, 0.15);
  padding: 2px 6px;
  border-radius: 8px;
  margin-left: 4px;
}

@keyframes comboPulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.75; transform: scale(1.05); }
}

.difficulty-indicator {
  margin-top: 4px;
}

.difficulty-text {
  font-size: 11px;
  color: rgba(255, 100, 150, 0.7);
  background: rgba(255, 100, 150, 0.1);
  padding: 2px 8px;
  border-radius: 10px;
}

.canvas-container {
  position: relative;
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 8px;
  overflow: hidden;
}

.game-canvas {
  border-radius: 12px;
  box-shadow: 
    0 8px 32px rgba(0, 0, 0, 0.35),
    inset 0 0 0 1px rgba(255, 255, 255, 0.06);
  cursor: pointer;
  touch-action: none;
  image-rendering: optimizeQuality;
}

.charge-overlay {
  position: absolute;
  top: 55%;
  left: 50%;
  transform: translate(-50%, -50%);
  display: flex;
  flex-direction: column;
  align-items: center;
  pointer-events: none;
  z-index: 15;
}

.circular-charge {
  position: relative;
  width: 70px;
  height: 70px;
}

.charge-svg {
  width: 70px;
  height: 70px;
  transform: rotate(-90deg);
}

.charge-ring-bg {
  fill: none;
  stroke: rgba(255, 255, 255, 0.08);
  stroke-width: 5;
  stroke-linecap: round;
}

.charge-ring {
  fill: none;
  stroke: #4ECDC4;
  stroke-width: 5;
  stroke-linecap: round;
  stroke-dasharray: 264;
  stroke-dashoffset: 264;
  transition: stroke-dashoffset 0.05s linear;
}

.charge-percentage {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  font-size: 16px;
  font-weight: 800;
  color: #fff;
  text-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
}

.charge-hint {
  margin-top: 10px;
  font-size: 13px;
  font-weight: 600;
  color: rgba(255, 255, 255, 0.9);
  animation: hintBounce 0.55s ease-in-out infinite;
  background: rgba(0, 0, 0, 0.3);
  padding: 6px 16px;
  border-radius: 20px;
  backdrop-filter: blur(4px);
}

@keyframes hintBounce {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-4px); }
}

.direction-indicator {
  position: absolute;
  top: 16px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 15;
}

.arrow {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 6px 14px;
  background: rgba(255, 255, 255, 0.08);
  border-radius: 20px;
  backdrop-filter: blur(8px);
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.arrow-icon {
  font-size: 16px;
  color: #4ECDC4;
  font-weight: bold;
}

.direction-text {
  font-size: 12px;
  font-weight: 600;
  color: rgba(255, 255, 255, 0.8);
}

.game-footer {
  padding: 8px 20px 16px;
  background: linear-gradient(
    to top,
    rgba(0, 0, 0, 0.35),
    transparent
  );
  z-index: 10;
  flex-shrink: 0;
}

.hint-text {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  color: rgba(255, 255, 255, 0.4);
}

.hint-icon {
  font-size: 14px;
}

.game-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(
    to bottom,
    rgba(8, 8, 18, 0.97),
    rgba(12, 12, 28, 0.98)
  );
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 30;
  backdrop-filter: blur(12px);
}

.overlay-content {
  text-align: center;
  padding: 0 24px;
  max-width: 100%;
  width: 100%;
}

.game-logo {
  margin-bottom: 16px;
}

.logo-icon {
  font-size: 56px;
  display: inline-block;
  animation: logoFloat 3s ease-in-out infinite;
}

@keyframes logoFloat {
  0%, 100% { transform: translateY(0) rotate(-6deg); }
  50% { transform: translateY(-12px) rotate(6deg); }
}

.game-title {
  font-size: 38px;
  font-weight: 800;
  color: #fff;
  margin: 0 0 6px;
  background: linear-gradient(135deg, #fff 0%, #b8b8ff 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  letter-spacing: 2px;
}

.game-desc {
  font-size: 13px;
  color: rgba(255, 255, 255, 0.4);
  margin: 0 0 24px;
}

.game-tutorial {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-bottom: 28px;
  max-width: 280px;
  margin-left: auto;
  margin-right: auto;
}

.tutorial-step {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  text-align: left;
  padding: 10px 12px;
  background: rgba(255, 255, 255, 0.03);
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.05);
}

.step-num {
  width: 26px;
  height: 26px;
  display: flex;
  justify-content: center;
  align-items: center;
  background: linear-gradient(135deg, #4ECDC4, #45B7D1);
  border-radius: 8px;
  font-size: 13px;
  font-weight: 700;
  color: #fff;
  flex-shrink: 0;
}

.step-content {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.step-title {
  font-size: 13px;
  font-weight: 600;
  color: rgba(255, 255, 255, 0.9);
}

.step-desc {
  font-size: 11px;
  color: rgba(255, 255, 255, 0.4);
  line-height: 1.4;
}

.quick-stats {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  margin-bottom: 24px;
  background: rgba(255, 255, 255, 0.03);
  padding: 8px 16px;
  border-radius: 20px;
  display: inline-flex;
}

.stat-label {
  font-size: 11px;
  color: rgba(255, 255, 255, 0.4);
}

.stat-value {
  font-size: 20px;
  font-weight: 700;
  color: #4ECDC4;
}

.start-btn, .restart-btn {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 14px 36px;
  font-size: 15px;
  font-weight: 700;
  color: #fff;
  background: linear-gradient(135deg, #4ECDC4 0%, #44A08D 100%);
  border: none;
  border-radius: 26px;
  cursor: pointer;
  transition: all 0.25s cubic-bezier(0.34, 1.56, 0.64, 1);
  box-shadow: 
    0 8px 24px rgba(78, 205, 196, 0.25),
    0 0 0 1px rgba(255, 255, 255, 0.08);
}

.start-btn:hover, .restart-btn:hover {
  transform: translateY(-3px) scale(1.03);
  box-shadow: 
    0 12px 32px rgba(78, 205, 196, 0.35),
    0 0 0 1px rgba(255, 255, 255, 0.12);
}

.start-btn:active, .restart-btn:active {
  transform: translateY(0) scale(0.97);
}

.btn-text {
  letter-spacing: 0.5px;
}

.btn-icon {
  font-size: 16px;
}

.result-icon {
  font-size: 52px;
  margin-bottom: 14px;
  display: inline-block;
}

.result-icon.new-record {
  animation: recordBounce 0.7s cubic-bezier(0.34, 1.56, 0.64, 1);
}

@keyframes recordBounce {
  0% { transform: scale(1) rotate(0deg); }
  30% { transform: scale(1.5) rotate(10deg); }
  60% { transform: scale(1.2) rotate(-8deg); }
  100% { transform: scale(1) rotate(0deg); }
}

.result-title {
  font-size: 26px;
  font-weight: 700;
  color: #fff;
  margin: 0 0 16px;
}

.final-score {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-bottom: 12px;
}

.final-label {
  font-size: 11px;
  color: rgba(255, 255, 255, 0.4);
  text-transform: uppercase;
  letter-spacing: 1.5px;
  margin-bottom: 4px;
}

.final-value {
  font-size: 52px;
  font-weight: 800;
  color: #fff;
  line-height: 1;
  text-shadow: 0 4px 24px rgba(255, 255, 255, 0.15);
  font-variant-numeric: tabular-nums;
}

.result-stats {
  display: flex;
  flex-direction: column;
  gap: 6px;
  margin-bottom: 28px;
  background: rgba(255, 255, 255, 0.03);
  padding: 12px 20px;
  border-radius: 12px;
  display: inline-block;
}

.stat-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 20px;
  min-width: 120px;
}

.stat-desc {
  font-size: 12px;
  color: rgba(255, 255, 255, 0.4);
}

.stat-num {
  font-size: 16px;
  font-weight: 700;
  color: #4ECDC4;
  font-variant-numeric: tabular-nums;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

@media (max-width: 520px) {
  .jump-game {
    border-radius: 0;
    margin: 0;
    width: 100%;
    height: 100%;
    max-width: 100%;
    max-height: 100%;
  }
  
  .game-title {
    font-size: 32px;
  }
  
  .final-value {
    font-size: 44px;
  }
  
  .score-value {
    font-size: 26px;
  }
  
  .game-tutorial {
    max-width: 100%;
  }
}

@media (max-height: 700px) {
  .game-header {
    padding: 10px 16px 6px;
  }
  
  .score-value {
    font-size: 24px;
  }
  
  .game-footer {
    padding: 6px 16px 12px;
  }
  
  .logo-icon {
    font-size: 44px;
  }
  
  .game-title {
    font-size: 28px;
  }
  
  .final-value {
    font-size: 40px;
  }
  
  .tutorial-step {
    padding: 8px 10px;
  }
}

@media (prefers-reduced-motion: reduce) {
  .logo-icon,
  .combo-text,
  .charge-hint,
  .score-animate,
  .result-icon.new-record {
    animation: none;
  }
  
  .start-btn:hover,
  .restart-btn:hover {
    transform: none;
  }
}
</style>
