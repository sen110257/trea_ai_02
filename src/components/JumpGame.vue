<template>
  <div class="game-wrapper">
    <div class="jump-game">
      <div class="game-header">
        <div class="score-display">
          <div class="score-item current-score">
            <span class="score-label">分数</span>
            <span class="score-value" :class="{ 'score-animate': scoreChanged }">{{ score }}</span>
          </div>
          <div class="score-item best-score">
            <span class="score-label">最高分</span>
            <span class="score-value">{{ bestScore }}</span>
          </div>
        </div>
        <div class="combo-display" v-if="combo > 1">
          <span class="combo-text">完美 x{{ combo }}</span>
        </div>
      </div>

      <div class="canvas-container">
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

        <div class="charge-overlay" v-if="gameState === 'CHARGING'">
          <div class="circular-charge">
            <svg class="charge-svg" viewBox="0 0 100 100">
              <circle class="charge-ring-bg" cx="50" cy="50" r="45"></circle>
              <circle 
                class="charge-ring" 
                cx="50" 
                cy="50" 
                r="45"
                :style="{ strokeDashoffset: 283 - (283 * chargePercent / 100) }"
              ></circle>
            </svg>
            <div class="charge-percentage">{{ Math.round(chargePercent) }}%</div>
          </div>
          <div class="charge-hint">松开跳跃！</div>
        </div>

        <div class="direction-indicator" v-if="gameState === 'CHARGING' && nextPlatform">
          <div class="arrow" :class="{ 'arrow-left': direction < 0, 'arrow-right': direction > 0 }">
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

      <div class="game-overlay" v-if="gameState === 'IDLE'">
        <div class="overlay-content">
          <div class="game-logo">
            <div class="logo-icon">🎯</div>
          </div>
          <h1 class="game-title">跳一跳</h1>
          <p class="game-desc">经典小游戏 · 挑战最高分</p>
          <div class="game-tips">
            <div class="tip-item">
              <span class="tip-num">1</span>
              <span>长按蓄力</span>
            </div>
            <div class="tip-item">
              <span class="tip-num">2</span>
              <span>观察方向</span>
            </div>
            <div class="tip-item">
              <span class="tip-num">3</span>
              <span>精准跳跃</span>
            </div>
          </div>
          <button class="start-btn" @click="startGame">
            <span class="btn-text">开始游戏</span>
            <span class="btn-icon">▶</span>
          </button>
        </div>
      </div>

      <div class="game-overlay" v-if="gameState === 'GAME_OVER'">
        <div class="overlay-content">
          <div class="result-icon" :class="{ 'new-record': score === bestScore && score > 0 }">
            <span v-if="score === bestScore && score > 0">🏆</span>
            <span v-else>💫</span>
          </div>
          <h1 class="result-title">
            {{ score === bestScore && score > 0 ? '新纪录！' : '游戏结束' }}
          </h1>
          <div class="final-score">
            <span class="final-label">最终分数</span>
            <span class="final-value">{{ score }}</span>
          </div>
          <div class="stats-info" v-if="bestScore > 0">
            <span>历史最高: {{ bestScore }}</span>
          </div>
          <button class="restart-btn" @click="startGame">
            <span class="btn-icon">🔄</span>
            <span class="btn-text">再来一局</span>
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed, watch, nextTick } from 'vue'

const canvasRef = ref(null)
const canvasWidth = 480
const canvasHeight = 640

const GRAVITY = 0.4
const MAX_CHARGE = 100
const CHARGE_SPEED = 1.2
const PLATFORM_MIN_WIDTH = 50
const PLATFORM_MAX_WIDTH = 90
const PLATFORM_HEIGHT = 25
const PLATFORM_MIN_GAP = 90
const PLATFORM_MAX_GAP = 160
const PLAYER_WIDTH = 24
const PLAYER_HEIGHT = 40

const gameState = ref('IDLE')
const score = ref(0)
const bestScore = ref(0)
const combo = ref(0)
const scoreChanged = ref(false)
const chargeTime = ref(0)
const chargePercent = computed(() => Math.min(100, (chargeTime.value / MAX_CHARGE) * 100))

let ctx = null
let animationId = null
let lastTime = 0

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
    x: Math.random() * canvasWidth,
    y: Math.random() * canvasHeight,
    size: Math.random() * 2 + 0.5,
    opacity: Math.random() * 0.5 + 0.2,
    twinkleSpeed: Math.random() * 0.02 + 0.01
  })
}

function initGame() {
  platforms.value = []
  
  const firstWidth = 100
  const firstPlatform = {
    x: canvasWidth / 2 - firstWidth / 2,
    y: canvasHeight - 180,
    width: firstWidth,
    height: PLATFORM_HEIGHT,
    color: platformColors[0],
    radius: 8
  }
  platforms.value.push(firstPlatform)
  
  for (let i = 1; i < 12; i++) {
    generatePlatform(i)
  }
  
  player.value = {
    x: canvasWidth / 2,
    y: firstPlatform.y,
    vx: 0,
    vy: 0,
    isJumping: false,
    currentPlatform: 0,
    squash: 1,
    stretch: 1
  }
  
  score.value = 0
  combo.value = 0
  chargeTime.value = 0
  particles.value = []
  landingEffect.value = { active: false, x: 0, y: 0, radius: 0, opacity: 1 }
}

function generatePlatform(index) {
  const prevPlatform = platforms.value[index - 1]
  const gap = PLATFORM_MIN_GAP + Math.random() * (PLATFORM_MAX_GAP - PLATFORM_MIN_GAP)
  const width = PLATFORM_MIN_WIDTH + Math.random() * (PLATFORM_MAX_WIDTH - PLATFORM_MIN_WIDTH)
  
  const direction = Math.random() > 0.5 ? 1 : -1
  const xOffset = (Math.random() * 60 + 40) * direction
  
  let newX = prevPlatform.x + prevPlatform.width / 2 + xOffset - width / 2
  newX = Math.max(30, Math.min(canvasWidth - width - 30, newX))
  
  platforms.value.push({
    x: newX,
    y: prevPlatform.y - gap,
    width: width,
    height: PLATFORM_HEIGHT,
    color: platformColors[index % platformColors.length],
    radius: 6 + Math.random() * 4
  })
}

function startGame() {
  initGame()
  gameState.value = 'PLAYING'
  lastTime = performance.now()
  startRender()
}

function startRender() {
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
  gameLoop()
}

function gameLoop(currentTime = performance.now()) {
  const deltaTime = (currentTime - lastTime) / 16.67
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
    p.x += p.vx
    p.y += p.vy
    p.vy += 0.1
    p.life -= 0.02
    p.opacity = p.life
    return p.life > 0
  })
  
  if (landingEffect.value.active) {
    landingEffect.value.radius += 3
    landingEffect.value.opacity -= 0.05
    if (landingEffect.value.opacity <= 0) {
      landingEffect.value.active = false
    }
  }
  
  backgroundStars.forEach(star => {
    star.opacity = 0.2 + Math.abs(Math.sin(performance.now() * star.twinkleSpeed)) * 0.5
  })
}

function updatePhysics(deltaTime) {
  const dt = Math.min(deltaTime, 2)
  
  player.value.x += player.value.vx * dt
  player.value.y += player.value.vy * dt
  player.value.vy += GRAVITY * dt
  
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
    
    if (playerBottom >= platformTop && playerBottom <= platformBottom + 10) {
      const playerLeft = p.x - PLAYER_WIDTH / 2
      const playerRight = p.x + PLAYER_WIDTH / 2
      const platformLeft = platform.x
      const platformRight = platform.x + platform.width
      
      const overlapLeft = Math.max(playerLeft, platformLeft)
      const overlapRight = Math.min(playerRight, platformRight)
      
      if (overlapRight > overlapLeft) {
        p.y = platformTop
        p.vy = 0
        p.vx = 0
        p.isJumping = false
        
        p.squash = 0.7
        p.stretch = 1.2
        
        createLandingEffect(p.x, platformTop)
        
        if (index !== p.currentPlatform) {
          p.currentPlatform = index
          
          const platformCenterX = platform.x + platform.width / 2
          const distanceFromCenter = Math.abs(p.x - platformCenterX)
          const maxDistance = platform.width / 2 - PLAYER_WIDTH / 2
          
          if (distanceFromCenter < 8) {
            combo.value++
            score.value += 1 + Math.floor(combo.value / 3)
          } else {
            combo.value = 1
            score.value += 1
          }
          
          if (score.value > bestScore.value) {
            bestScore.value = score.value
          }
          
          scoreChanged.value = true
          setTimeout(() => { scoreChanged.value = false }, 300)
          
          generatePlatform(platforms.value.length)
          
          smoothAdjustView()
        }
        
        return
      }
    }
  }
  
  if (p.y > canvasHeight + 200) {
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
  
  for (let i = 0; i < 8; i++) {
    const angle = (Math.PI * 2 * i) / 8 + Math.random() * 0.5
    const speed = 2 + Math.random() * 3
    particles.value.push({
      x: x,
      y: y,
      vx: Math.cos(angle) * speed,
      vy: -Math.random() * 3 - 1,
      size: 3 + Math.random() * 3,
      color: `hsl(${Math.random() * 60 + 0}, 100%, 70%)`,
      life: 1,
      opacity: 1
    })
  }
}

function smoothAdjustView() {
  const currentPlatform = platforms.value[player.value.currentPlatform]
  const targetY = canvasHeight - 220
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
  const gradient = ctx.createLinearGradient(0, 0, 0, canvasHeight)
  gradient.addColorStop(0, '#1a1a2e')
  gradient.addColorStop(0.5, '#16213e')
  gradient.addColorStop(1, '#0f0f23')
  ctx.fillStyle = gradient
  ctx.fillRect(0, 0, canvasWidth, canvasHeight)
  
  backgroundStars.forEach(star => {
    ctx.fillStyle = `rgba(255, 255, 255, ${star.opacity})`
    ctx.beginPath()
    ctx.arc(star.x, star.y, star.size, 0, Math.PI * 2)
    ctx.fill()
  })
  
  ctx.strokeStyle = 'rgba(255, 255, 255, 0.03)'
  ctx.lineWidth = 1
  
  for (let x = 0; x < canvasWidth; x += 40) {
    ctx.beginPath()
    ctx.moveTo(x, 0)
    ctx.lineTo(x, canvasHeight)
    ctx.stroke()
  }
  
  for (let y = 0; y < canvasHeight; y += 40) {
    ctx.beginPath()
    ctx.moveTo(0, y)
    ctx.lineTo(canvasWidth, y)
    ctx.stroke()
  }
}

function drawPlatforms() {
  platforms.value.forEach((platform, index) => {
    const isCurrentPlatform = index === player.value.currentPlatform && !player.value.isJumping
    const { x, y, width, height, color, radius } = platform
    
    ctx.fillStyle = 'rgba(0, 0, 0, 0.3)'
    roundRect(ctx, x + 4, y + 4, width, height, radius)
    ctx.fill()
    
    const bodyGradient = ctx.createLinearGradient(x, y, x, y + height)
    bodyGradient.addColorStop(0, color.light)
    bodyGradient.addColorStop(0.5, color.main)
    bodyGradient.addColorStop(1, color.dark)
    ctx.fillStyle = bodyGradient
    roundRect(ctx, x, y, width, height, radius)
    ctx.fill()
    
    ctx.fillStyle = 'rgba(255, 255, 255, 0.4)'
    roundRect(ctx, x + 2, y + 2, width - 4, height / 3, radius - 2)
    ctx.fill()
    
    if (isCurrentPlatform) {
      ctx.strokeStyle = 'rgba(255, 255, 255, 0.6)'
      ctx.lineWidth = 2
      roundRect(ctx, x - 1, y - 1, width + 2, height + 2, radius + 1)
      ctx.stroke()
      
      const centerX = x + width / 2
      ctx.fillStyle = 'rgba(255, 255, 255, 0.8)'
      ctx.beginPath()
      ctx.arc(centerX, y - 8, 3, 0, Math.PI * 2)
      ctx.fill()
    }
    
    if (index === player.value.currentPlatform + 1 && gameState.value === 'PLAYING') {
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
  ctx.ellipse(2, PLAYER_HEIGHT / 2 + 5, PLAYER_WIDTH / 2, 5, 0, 0, Math.PI * 2)
  ctx.fill()
  
  const bodyGradient = ctx.createLinearGradient(
    -PLAYER_WIDTH / 2, -PLAYER_HEIGHT,
    PLAYER_WIDTH / 2, 0
  )
  bodyGradient.addColorStop(0, '#FF6B6B')
  bodyGradient.addColorStop(0.5, '#EE5A5A')
  bodyGradient.addColorStop(1, '#C0392B')
  ctx.fillStyle = bodyGradient
  
  ctx.beginPath()
  ctx.moveTo(-PLAYER_WIDTH / 2 + 4, 0)
  ctx.quadraticCurveTo(-PLAYER_WIDTH / 2, 0, -PLAYER_WIDTH / 2, -4)
  ctx.lineTo(-PLAYER_WIDTH / 2, -PLAYER_HEIGHT + 6)
  ctx.quadraticCurveTo(-PLAYER_WIDTH / 2, -PLAYER_HEIGHT, -PLAYER_WIDTH / 2 + 6, -PLAYER_HEIGHT)
  ctx.lineTo(PLAYER_WIDTH / 2 - 6, -PLAYER_HEIGHT)
  ctx.quadraticCurveTo(PLAYER_WIDTH / 2, -PLAYER_HEIGHT, PLAYER_WIDTH / 2, -PLAYER_HEIGHT + 6)
  ctx.lineTo(PLAYER_WIDTH / 2, -4)
  ctx.quadraticCurveTo(PLAYER_WIDTH / 2, 0, PLAYER_WIDTH / 2 - 4, 0)
  ctx.closePath()
  ctx.fill()
  
  ctx.fillStyle = 'rgba(255, 255, 255, 0.3)'
  ctx.beginPath()
  ctx.ellipse(-PLAYER_WIDTH / 4, -PLAYER_HEIGHT / 2, 3, PLAYER_HEIGHT / 3, 0, 0, Math.PI * 2)
  ctx.fill()
  
  const eyeOffsetX = 5
  const eyeOffsetY = -PLAYER_HEIGHT / 2 - 4
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
  ctx.ellipse(-eyeOffsetX - 4, eyeOffsetY + 5, 3, 2, 0, 0, Math.PI * 2)
  ctx.ellipse(eyeOffsetX + 4, eyeOffsetY + 5, 3, 2, 0, 0, Math.PI * 2)
  ctx.fill()
  
  if (gameState.value === 'CHARGING') {
    const pulseSize = 15 + chargePercent.value / 8
    const pulseOpacity = 0.2 + chargePercent.value / 200
    
    ctx.fillStyle = `rgba(255, 255, 255, ${pulseOpacity})`
    ctx.beginPath()
    ctx.arc(0, -PLAYER_HEIGHT / 2, pulseSize, 0, Math.PI * 2)
    ctx.fill()
    
    for (let i = 0; i < 4; i++) {
      const particleAngle = (Math.PI / 2) + (i - 1.5) * 0.5
      const particleDist = pulseSize + 5 + Math.sin(performance.now() * 0.01 + i) * 3
      const px = Math.cos(particleAngle) * particleDist
      const py = Math.sin(particleAngle) * particleDist - PLAYER_HEIGHT / 2
      
      ctx.fillStyle = `rgba(255, 255, 255, ${0.5 + chargePercent.value / 200})`
      ctx.beginPath()
      ctx.arc(px, py, 2 + chargePercent.value / 50, 0, Math.PI * 2)
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
  
  ctx.strokeStyle = `rgba(255, 200, 100, ${opacity * 0.5})`
  ctx.lineWidth = 3
  ctx.beginPath()
  ctx.arc(x, y, radius * 0.6, 0, Math.PI * 2)
  ctx.stroke()
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
  ctx.beginPath()
  ctx.moveTo(x + radius, y)
  ctx.lineTo(x + width - radius, y)
  ctx.quadraticCurveTo(x + width, y, x + width, y + radius)
  ctx.lineTo(x + width, y + height - radius)
  ctx.quadraticCurveTo(x + width, y + height, x + width - radius, y + height)
  ctx.lineTo(x + radius, y + height)
  ctx.quadraticCurveTo(x, y + height, x, y + height - radius)
  ctx.lineTo(x, y + radius)
  ctx.quadraticCurveTo(x, y, x + radius, y)
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
      if (chargeTime.value < MAX_CHARGE) {
        chargeTime.value += CHARGE_SPEED
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
  
  const jumpVx = (power * 7 + 1) * dir
  const jumpVy = -(power * 10 + 5)
  
  player.value.vx = jumpVx
  player.value.vy = jumpVy
  player.value.isJumping = true
  player.value.squash = 1.2
  player.value.stretch = 0.8
  
  gameState.value = 'PLAYING'
  chargeTime.value = 0
}

function gameOver() {
  player.value.isJumping = false
  gameState.value = 'GAME_OVER'
  
  if (animationId) {
    cancelAnimationFrame(animationId)
    animationId = null
  }
}

onMounted(() => {
  if (canvasRef.value) {
    ctx = canvasRef.value.getContext('2d')
    ctx.imageSmoothingEnabled = true
    ctx.imageSmoothingQuality = 'high'
    
    initGame()
    
    gameState.value = 'IDLE'
    render()
  }
})

onUnmounted(() => {
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
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
}

.jump-game {
  position: relative;
  width: 100%;
  max-width: 520px;
  height: 100%;
  max-height: 800px;
  display: flex;
  flex-direction: column;
  background: rgba(20, 20, 40, 0.95);
  border-radius: 20px;
  box-shadow: 
    0 25px 80px rgba(0, 0, 0, 0.5),
    0 0 0 1px rgba(255, 255, 255, 0.05),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
  overflow: hidden;
}

.game-header {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 16px 20px 8px;
  background: linear-gradient(
    to bottom,
    rgba(0, 0, 0, 0.4),
    rgba(0, 0, 0, 0.2),
    transparent
  );
  z-index: 10;
}

.score-display {
  display: flex;
  justify-content: center;
  gap: 40px;
  width: 100%;
}

.score-item {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.score-label {
  font-size: 11px;
  color: rgba(255, 255, 255, 0.5);
  text-transform: uppercase;
  letter-spacing: 1px;
  margin-bottom: 2px;
}

.score-value {
  font-size: 32px;
  font-weight: 700;
  color: #fff;
  text-shadow: 0 2px 10px rgba(255, 255, 255, 0.2);
  line-height: 1;
}

.score-animate {
  animation: scorePop 0.3s ease-out;
}

@keyframes scorePop {
  0% { transform: scale(1); }
  50% { transform: scale(1.3); color: #FFD700; }
  100% { transform: scale(1); }
}

.combo-display {
  margin-top: 8px;
}

.combo-text {
  font-size: 14px;
  font-weight: 600;
  color: #FFD700;
  text-shadow: 0 0 10px rgba(255, 215, 0, 0.5);
  animation: comboPulse 0.5s ease-in-out infinite;
}

@keyframes comboPulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.7; }
}

.canvas-container {
  position: relative;
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 0 20px;
}

.game-canvas {
  border-radius: 12px;
  box-shadow: 
    0 8px 32px rgba(0, 0, 0, 0.3),
    inset 0 0 0 1px rgba(255, 255, 255, 0.05);
  cursor: pointer;
  touch-action: none;
  max-width: 100%;
  max-height: 100%;
}

.charge-overlay {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  display: flex;
  flex-direction: column;
  align-items: center;
  pointer-events: none;
  z-index: 15;
  margin-top: 80px;
}

.circular-charge {
  position: relative;
  width: 80px;
  height: 80px;
}

.charge-svg {
  width: 80px;
  height: 80px;
  transform: rotate(-90deg);
}

.charge-ring-bg {
  fill: none;
  stroke: rgba(255, 255, 255, 0.1);
  stroke-width: 6;
  stroke-linecap: round;
}

.charge-ring {
  fill: none;
  stroke: linear-gradient(90deg, #4ECDC4, #45B7D1);
  stroke: #4ECDC4;
  stroke-width: 6;
  stroke-linecap: round;
  stroke-dasharray: 283;
  stroke-dashoffset: 283;
  transition: stroke-dashoffset 0.05s linear;
}

.charge-percentage {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  font-size: 16px;
  font-weight: 700;
  color: #fff;
}

.charge-hint {
  margin-top: 12px;
  font-size: 14px;
  font-weight: 600;
  color: rgba(255, 255, 255, 0.9);
  animation: hintBounce 0.6s ease-in-out infinite;
}

@keyframes hintBounce {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-3px); }
}

.direction-indicator {
  position: absolute;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 15;
}

.arrow {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 16px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 20px;
  backdrop-filter: blur(10px);
}

.arrow-left::before {
  content: '←';
  font-size: 18px;
  color: #4ECDC4;
}

.arrow-right::after {
  content: '→';
  font-size: 18px;
  color: #4ECDC4;
}

.direction-text {
  font-size: 13px;
  font-weight: 600;
  color: rgba(255, 255, 255, 0.8);
}

.game-footer {
  padding: 12px 20px 20px;
  background: linear-gradient(
    to top,
    rgba(0, 0, 0, 0.3),
    transparent
  );
  z-index: 10;
}

.hint-text {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  color: rgba(255, 255, 255, 0.5);
}

.hint-icon {
  font-size: 16px;
}

.game-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(
    to bottom,
    rgba(10, 10, 20, 0.95),
    rgba(15, 15, 35, 0.98)
  );
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 30;
  backdrop-filter: blur(8px);
}

.overlay-content {
  text-align: center;
  padding: 0 40px;
  max-width: 100%;
}

.game-logo {
  margin-bottom: 20px;
}

.logo-icon {
  font-size: 64px;
  display: inline-block;
  animation: logoFloat 3s ease-in-out infinite;
}

@keyframes logoFloat {
  0%, 100% { transform: translateY(0) rotate(-5deg); }
  50% { transform: translateY(-10px) rotate(5deg); }
}

.game-title {
  font-size: 42px;
  font-weight: 800;
  color: #fff;
  margin: 0 0 8px;
  background: linear-gradient(135deg, #fff 0%, #a0a0ff 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.game-desc {
  font-size: 14px;
  color: rgba(255, 255, 255, 0.5);
  margin: 0 0 32px;
}

.game-tips {
  display: flex;
  justify-content: center;
  gap: 24px;
  margin-bottom: 40px;
}

.tip-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
}

.tip-num {
  width: 28px;
  height: 28px;
  display: flex;
  justify-content: center;
  align-items: center;
  background: linear-gradient(135deg, #4ECDC4, #45B7D1);
  border-radius: 50%;
  font-size: 14px;
  font-weight: 700;
  color: #fff;
}

.tip-item span:last-child {
  font-size: 12px;
  color: rgba(255, 255, 255, 0.6);
}

.result-icon {
  font-size: 56px;
  margin-bottom: 16px;
}

.result-icon.new-record {
  animation: recordBounce 0.6s ease-out;
}

@keyframes recordBounce {
  0% { transform: scale(1); }
  50% { transform: scale(1.4); }
  100% { transform: scale(1); }
}

.result-title {
  font-size: 28px;
  font-weight: 700;
  color: #fff;
  margin: 0 0 20px;
}

.final-score {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-bottom: 12px;
}

.final-label {
  font-size: 12px;
  color: rgba(255, 255, 255, 0.5);
  text-transform: uppercase;
  letter-spacing: 1px;
  margin-bottom: 4px;
}

.final-value {
  font-size: 56px;
  font-weight: 800;
  color: #fff;
  line-height: 1;
  text-shadow: 0 4px 20px rgba(255, 255, 255, 0.2);
}

.stats-info {
  font-size: 13px;
  color: rgba(255, 255, 255, 0.5);
  margin-bottom: 32px;
}

.start-btn, .restart-btn {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  padding: 16px 40px;
  font-size: 16px;
  font-weight: 700;
  color: #fff;
  background: linear-gradient(135deg, #4ECDC4 0%, #44A08D 100%);
  border: none;
  border-radius: 30px;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 
    0 8px 25px rgba(78, 205, 196, 0.3),
    0 0 0 1px rgba(255, 255, 255, 0.1);
}

.start-btn:hover, .restart-btn:hover {
  transform: translateY(-2px) scale(1.02);
  box-shadow: 
    0 12px 35px rgba(78, 205, 196, 0.4),
    0 0 0 1px rgba(255, 255, 255, 0.15);
}

.start-btn:active, .restart-btn:active {
  transform: translateY(0) scale(0.98);
}

.btn-text {
  letter-spacing: 0.5px;
}

.btn-icon {
  font-size: 18px;
}

@media (max-width: 520px) {
  .jump-game {
    border-radius: 0;
    max-height: 100%;
  }
  
  .game-title {
    font-size: 32px;
  }
  
  .final-value {
    font-size: 44px;
  }
  
  .game-tips {
    gap: 16px;
  }
}
</style>
