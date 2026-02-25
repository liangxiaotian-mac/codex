<template>
  <main class="page">
    <h1>✈️ 飞机大战：重制版</h1>
    <p class="tip">WASD/方向键移动，空格射击，K 使用炸弹，P 暂停，R 重开。</p>

    <section class="hud">
      <span>分数：{{ score }}</span>
      <span>生命：{{ playerHp }}</span>
      <span>等级：Lv.{{ level }}</span>
      <span>炸弹：{{ bombs }}</span>
      <span>最高分：{{ bestScore }}</span>
    </section>

    <div ref="gameAreaRef" class="game-area" tabindex="0" @click="focusArea">
      <div class="stars"></div>

      <div class="player" :style="playerStyle">▲</div>

      <div
        v-for="bullet in bullets"
        :key="bullet.id"
        class="bullet"
        :style="{ left: `${bullet.x}px`, top: `${bullet.y}px` }"
      ></div>

      <div
        v-for="enemy in enemies"
        :key="enemy.id"
        class="enemy"
        :class="enemy.type"
        :style="{ left: `${enemy.x}px`, top: `${enemy.y}px` }"
      >
        {{ enemy.type === 'heavy' ? '◆' : '▼' }}
      </div>

      <div
        v-for="flash in flashes"
        :key="flash.id"
        class="flash"
        :style="{ left: `${flash.x}px`, top: `${flash.y}px`, opacity: flash.life / 18 }"
      ></div>

      <div v-if="status !== 'running'" class="overlay">
        <template v-if="status === 'ready'">
          <h2>准备起飞</h2>
          <p>点击下方按钮开始游戏</p>
          <button @click="startGame">开始游戏</button>
        </template>

        <template v-else-if="status === 'paused'">
          <h2>已暂停</h2>
          <p>按 P 继续，或点击按钮继续</p>
          <button @click="resumeGame">继续游戏</button>
        </template>

        <template v-else>
          <h2>游戏结束</h2>
          <p>你的分数：{{ score }}</p>
          <button @click="restart">再来一局</button>
        </template>
      </div>
    </div>

    <section class="mobile-controls" aria-label="mobile controls">
      <div class="row">
        <button @touchstart.prevent="press('ArrowUp')" @touchend.prevent="release('ArrowUp')">↑</button>
      </div>
      <div class="row">
        <button @touchstart.prevent="press('ArrowLeft')" @touchend.prevent="release('ArrowLeft')">←</button>
        <button @touchstart.prevent="press('Space')" @touchend.prevent="release('Space')">开火</button>
        <button @touchstart.prevent="press('ArrowRight')" @touchend.prevent="release('ArrowRight')">→</button>
      </div>
      <div class="row">
        <button @touchstart.prevent="press('ArrowDown')" @touchend.prevent="release('ArrowDown')">↓</button>
        <button @click="triggerBomb">炸弹(K)</button>
      </div>
    </section>
  </main>
</template>

<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

const width = 420
const height = 640
const playerSize = 28

const gameAreaRef = ref(null)
const score = ref(0)
const bestScore = ref(Number(localStorage.getItem('plane-best-score') || 0))
const playerHp = ref(4)
const bombs = ref(2)
const level = ref(1)
const status = ref('ready') // ready | running | paused | over

const player = ref({ x: width / 2 - playerSize / 2, y: height - 70, speed: 5 })
const bullets = ref([])
const enemies = ref([])
const flashes = ref([])

const keys = new Set()
let bulletId = 0
let enemyId = 0
let flashId = 0
let enemySpawnTimer = 0
let shootCooldown = 0
let invincibleTimer = 0
let loopId = 0

const playerStyle = computed(() => ({
  left: `${player.value.x}px`,
  top: `${player.value.y}px`,
  opacity: invincibleTimer > 0 ? 0.45 + Math.sin(Date.now() / 40) * 0.25 : 1,
}))

const focusArea = () => gameAreaRef.value?.focus()

const press = (code) => keys.add(code)
const release = (code) => keys.delete(code)

const onKeyDown = (e) => {
  if (['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight', 'Space'].includes(e.code)) {
    e.preventDefault()
  }

  keys.add(e.code)

  if (e.code === 'KeyR') restart()
  if (e.code === 'KeyP') {
    if (status.value === 'running') pauseGame()
    else if (status.value === 'paused') resumeGame()
  }
  if (e.code === 'KeyK') triggerBomb()
}

const onKeyUp = (e) => {
  keys.delete(e.code)
}

const getLevelByScore = () => 1 + Math.min(9, Math.floor(score.value / 160))

const randomEnemy = () => {
  const heavyChance = Math.min(0.08 + score.value / 2500, 0.33)
  const heavy = Math.random() < heavyChance
  const size = heavy ? 38 : 28
  return {
    id: enemyId++,
    type: heavy ? 'heavy' : 'normal',
    hp: heavy ? 3 : 1,
    score: heavy ? 35 : 10,
    size,
    x: Math.random() * (width - size),
    y: -40,
    speed: (heavy ? 1.1 : 1.8) + Math.random() * 1.5 + level.value * 0.16,
  }
}

const spawnFlash = (x, y) => {
  flashes.value.push({ id: flashId++, x, y, life: 18 })
}

const shoot = () => {
  const fireLevel = 1 + Math.min(2, Math.floor(score.value / 220))

  if (fireLevel === 1) {
    bullets.value.push({ id: bulletId++, x: player.value.x + 11, y: player.value.y - 10, speed: 8 })
    return
  }

  if (fireLevel === 2) {
    bullets.value.push({ id: bulletId++, x: player.value.x + 5, y: player.value.y - 8, speed: 8.6 })
    bullets.value.push({ id: bulletId++, x: player.value.x + 16, y: player.value.y - 8, speed: 8.6 })
    return
  }

  bullets.value.push({ id: bulletId++, x: player.value.x + 3, y: player.value.y - 8, speed: 9 })
  bullets.value.push({ id: bulletId++, x: player.value.x + 11, y: player.value.y - 12, speed: 9.2 })
  bullets.value.push({ id: bulletId++, x: player.value.x + 19, y: player.value.y - 8, speed: 9 })
}

const updatePlayer = () => {
  if (keys.has('ArrowLeft') || keys.has('KeyA')) player.value.x -= player.value.speed
  if (keys.has('ArrowRight') || keys.has('KeyD')) player.value.x += player.value.speed
  if (keys.has('ArrowUp') || keys.has('KeyW')) player.value.y -= player.value.speed
  if (keys.has('ArrowDown') || keys.has('KeyS')) player.value.y += player.value.speed

  player.value.x = Math.max(0, Math.min(width - playerSize, player.value.x))
  player.value.y = Math.max(0, Math.min(height - playerSize, player.value.y))

  if ((keys.has('Space') || keys.has('KeyJ')) && shootCooldown <= 0) {
    shoot()
    const cooldownBase = 10 - Math.min(4, Math.floor(score.value / 260))
    shootCooldown = Math.max(4, cooldownBase)
  }

  if (shootCooldown > 0) shootCooldown--
  if (invincibleTimer > 0) invincibleTimer--
}

const intersects = (a, aw, ah, b, bw, bh) =>
  a.x < b.x + bw && a.x + aw > b.x && a.y < b.y + bh && a.y + ah > b.y

const hitPlayer = () => {
  if (invincibleTimer > 0) return

  playerHp.value--
  invincibleTimer = 60
  spawnFlash(player.value.x + 10, player.value.y + 8)

  if (playerHp.value <= 0) endGame()
}

const updateObjects = () => {
  bullets.value = bullets.value.map((b) => ({ ...b, y: b.y - b.speed })).filter((b) => b.y > -30)

  enemies.value = enemies.value
    .map((en) => ({ ...en, y: en.y + en.speed }))
    .filter((en) => en.y < height + 60)

  flashes.value = flashes.value.map((f) => ({ ...f, life: f.life - 1 })).filter((f) => f.life > 0)

  const bulletToRemove = new Set()

  for (const bullet of bullets.value) {
    for (const enemy of enemies.value) {
      if (intersects(bullet, 6, 14, enemy, enemy.size, enemy.size)) {
        bulletToRemove.add(bullet.id)
        enemy.hp--
        if (enemy.hp <= 0) {
          score.value += enemy.score
          spawnFlash(enemy.x + enemy.size / 2 - 6, enemy.y + enemy.size / 2 - 6)
        }
        break
      }
    }
  }

  bullets.value = bullets.value.filter((b) => !bulletToRemove.has(b.id))
  enemies.value = enemies.value.filter((e) => e.hp > 0)

  for (const enemy of enemies.value) {
    if (intersects(player.value, playerSize, playerSize, enemy, enemy.size, enemy.size)) {
      enemy.hp = 0
      hitPlayer()
    }
  }

  const leaks = enemies.value.filter((e) => e.y + e.size >= height - 8).length
  if (leaks > 0) {
    for (let i = 0; i < leaks; i++) hitPlayer()
  }

  enemies.value = enemies.value.filter((e) => e.hp > 0 && e.y + e.size < height - 8)
  level.value = getLevelByScore()
}

const spawnEnemies = () => {
  if (enemySpawnTimer > 0) {
    enemySpawnTimer--
    return
  }

  enemies.value.push(randomEnemy())
  if (level.value >= 4 && Math.random() > 0.55) enemies.value.push(randomEnemy())
  if (level.value >= 7 && Math.random() > 0.7) enemies.value.push(randomEnemy())

  enemySpawnTimer = Math.max(14, 44 - level.value * 3)
}

const triggerBomb = () => {
  if (status.value !== 'running') return
  if (bombs.value <= 0) return

  bombs.value--
  const cleared = enemies.value.length
  score.value += cleared * 8

  for (const enemy of enemies.value) {
    spawnFlash(enemy.x + enemy.size / 2 - 6, enemy.y + enemy.size / 2 - 6)
  }
  enemies.value = []
}

const startGame = () => {
  if (status.value !== 'ready') return
  status.value = 'running'
  focusArea()
}

const pauseGame = () => {
  if (status.value === 'running') status.value = 'paused'
}

const resumeGame = () => {
  if (status.value !== 'paused') return
  status.value = 'running'
  focusArea()
}

const endGame = () => {
  status.value = 'over'
  keys.clear()

  if (score.value > bestScore.value) {
    bestScore.value = score.value
    localStorage.setItem('plane-best-score', String(score.value))
  }
}

const loop = () => {
  if (status.value === 'running') {
    updatePlayer()
    spawnEnemies()
    updateObjects()
  }
  loopId = requestAnimationFrame(loop)
}

const restart = () => {
  score.value = 0
  playerHp.value = 4
  bombs.value = 2
  level.value = 1
  bullets.value = []
  enemies.value = []
  flashes.value = []
  enemySpawnTimer = 0
  shootCooldown = 0
  invincibleTimer = 0
  player.value = { x: width / 2 - playerSize / 2, y: height - 70, speed: 5 }
  status.value = 'running'
  focusArea()
}

onMounted(() => {
  window.addEventListener('keydown', onKeyDown, { passive: false })
  window.addEventListener('keyup', onKeyUp)
  focusArea()
  loop()
})

onBeforeUnmount(() => {
  window.removeEventListener('keydown', onKeyDown)
  window.removeEventListener('keyup', onKeyUp)
  cancelAnimationFrame(loopId)
})
</script>
