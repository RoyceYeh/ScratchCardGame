<script setup>
import { ref, reactive, onMounted, onUnmounted, nextTick } from 'vue'
import * as PIXI from 'pixi.js'

const prizeImages = ['/imgs/prize1.png', '/imgs/prize2.png', '/imgs/prize3.png', '/imgs/prize4.png', '/imgs/prize5.png', '/imgs/prize6.png']

const currentPrize = ref(prizeImages[0])
const scratchedArea = ref(0)
const scratchProgress = ref(0)
const isFinished = ref(false)
const root = ref(null)

// 響應式相關變數
const screenWidth = ref(window.innerWidth)
const screenHeight = ref(window.innerHeight)
const gameSize = ref({ width: 600, height: 400 })

let app, cover, texture, renderTexture, renderTextureSprite
let resizeObserver = null

// 計算遊戲畫布尺寸的函數
const calculateGameSize = () => {
  const containerPadding = 40 // 左右各20px padding
  const maxWidth = Math.min(screenWidth.value - containerPadding, 600)
  const maxHeight = Math.min(screenHeight.value * 0.5, 400)

  // 保持16:10的比例
  const aspectRatio = 600 / 400
  let width = maxWidth
  let height = width / aspectRatio

  if (height > maxHeight) {
    height = maxHeight
    width = height * aspectRatio
  }

  gameSize.value = { width: Math.floor(width), height: Math.floor(height) }
}

// 計算刷子大小（根據畫布大小調整）
const getBrushSize = () => {
  const baseSize = 50
  const scale = Math.min(gameSize.value.width / 600, gameSize.value.height / 400)
  return Math.max(baseSize * scale, 20) // 最小20px
}

// 計算線條寬度
const getLineWidth = () => {
  const baseWidth = 100
  const scale = Math.min(gameSize.value.width / 600, gameSize.value.height / 400)
  return Math.max(baseWidth * scale, 40) // 最小40px
}

// 視窗大小改變處理
const handleResize = () => {
  screenWidth.value = window.innerWidth
  screenHeight.value = window.innerHeight
  const oldSize = { ...gameSize.value }
  calculateGameSize()

  if (app && (oldSize.width !== gameSize.value.width || oldSize.height !== gameSize.value.height)) {
    // 儲存當前狀態
    const coverAlpha = cover ? cover.alpha : 1
    const currentScratchedArea = scratchedArea.value
    const currentProgress = scratchProgress.value

    // 重新調整 PIXI 應用程式大小
    app.renderer.resize(gameSize.value.width, gameSize.value.height)

    // 重新調整所有 sprites 大小
    if (cover) {
      cover.width = gameSize.value.width
      cover.height = gameSize.value.height
      cover.alpha = coverAlpha
    }

    if (texture) {
      texture.width = gameSize.value.width
      texture.height = gameSize.value.height
    }

    // 處理 renderTexture
    if (renderTexture) {
      const oldRenderTexture = renderTexture

      // 創建新的 renderTexture
      renderTexture = PIXI.RenderTexture.create(gameSize.value)
      renderTextureSprite.texture = renderTexture

      // 根據遊戲狀態處理
      if (isFinished.value) {
        // 遊戲已完成：直接移除 mask 顯示完整獎品
        texture.mask = null
        if (cover) cover.alpha = 0
      } else if (currentProgress > 0) {
        // 遊戲進行中且有刮除進度：嘗試按比例恢復
        texture.mask = renderTextureSprite

        try {
          // 創建一個臨時的縮放 sprite 來恢復內容
          const tempSprite = new PIXI.Sprite(oldRenderTexture)
          tempSprite.width = gameSize.value.width
          tempSprite.height = gameSize.value.height

          // 將縮放後的內容渲染到新的 renderTexture
          app.renderer.render({
            container: tempSprite,
            target: renderTexture,
            clear: true,
            skipUpdateTransform: false,
          })

          // 按比例調整刮除面積
          const sizeScale = (gameSize.value.width * gameSize.value.height) / (oldSize.width * oldSize.height)
          scratchedArea.value = currentScratchedArea * sizeScale
          scratchProgress.value = scratchedArea.value / (gameSize.value.width * gameSize.value.height)

          // 檢查是否達到完成條件
          if (scratchProgress.value >= 0.6) {
            isFinished.value = true
            texture.mask = null
            if (cover) cover.alpha = 0
          }
        } catch (error) {
          console.warn('無法恢復刮除內容，重置遊戲狀態:', error)
          // 如果恢復失敗，清空刮除狀態但保持 mask
          texture.mask = renderTextureSprite
          scratchedArea.value = 0
          scratchProgress.value = 0
        }
      } else {
        // 遊戲剛開始，沒有刮除內容
        texture.mask = renderTextureSprite
      }

      // 清理舊的 renderTexture
      oldRenderTexture.destroy()
    }

    // 更新 hitArea 以匹配新的螢幕大小
    if (app.stage) {
      app.stage.hitArea = app.screen
    }
  }
}

const restartGame = async () => {
  scratchedArea.value = 0
  scratchProgress.value = 0
  isFinished.value = false

  const randomIndex = Math.floor(Math.random() * prizeImages.length)
  currentPrize.value = prizeImages[randomIndex]

  if (app && cover && texture && renderTexture) {
    app.renderer.render({
      container: new PIXI.Container(),
      target: renderTexture,
      clear: true,
    })

    cover.alpha = 1
    texture.mask = renderTextureSprite
    await PIXI.Assets.load([currentPrize.value])
    texture.texture = PIXI.Texture.from(currentPrize.value)
  }
}

const initGame = async () => {
  // 初始化時計算遊戲尺寸
  calculateGameSize()

  const randomIndex = Math.floor(Math.random() * prizeImages.length)
  currentPrize.value = prizeImages[randomIndex]

  app = new PIXI.Application()

  await app.init({
    width: gameSize.value.width,
    height: gameSize.value.height,
    backgroundColor: 'transparent',
    antialias: true,
    resolution: window.devicePixelRatio || 1,
    autoDensity: true,
  })

  root.value.appendChild(app.canvas)

  const brushSize = getBrushSize()
  const lineWidth = getLineWidth()

  const brush = new PIXI.Graphics().circle(0, 0, brushSize).fill({ color: 0xffffff })
  const line = new PIXI.Graphics()

  await PIXI.Assets.load(['imgs/cover.png', currentPrize.value])

  const stageSize = gameSize.value

  cover = new PIXI.Sprite(PIXI.Texture.from('imgs/cover.png'))
  cover.width = stageSize.width
  cover.height = stageSize.height

  texture = new PIXI.Sprite(PIXI.Texture.from(currentPrize.value))
  texture.width = stageSize.width
  texture.height = stageSize.height

  renderTexture = PIXI.RenderTexture.create(stageSize)
  renderTextureSprite = new PIXI.Sprite(renderTexture)

  texture.mask = renderTextureSprite

  app.stage.addChild(cover, texture, renderTextureSprite)

  app.stage.eventMode = 'static'
  app.stage.hitArea = app.screen
  app.stage.on('pointerdown', pointerDown).on('pointerup', pointerUp).on('pointerupoutside', pointerUp).on('pointermove', pointerMove)

  let dragging = false
  let lastDrawnPoint = null

  function pointerMove({ global: { x, y } }) {
    if (dragging) {
      // 獲取當前的刷子和線條大小
      const currentBrushSize = getBrushSize()
      const currentLineWidth = getLineWidth()

      brush.clear().circle(0, 0, currentBrushSize).fill({ color: 0xffffff })
      brush.position.set(x, y)

      app.renderer.render({
        container: brush,
        target: renderTexture,
        clear: false,
        skipUpdateTransform: false,
      })

      if (lastDrawnPoint) {
        line.clear().moveTo(lastDrawnPoint.x, lastDrawnPoint.y).lineTo(x, y).stroke({ width: currentLineWidth, color: 0xffffff })

        app.renderer.render({
          container: line,
          target: renderTexture,
          clear: false,
          skipUpdateTransform: false,
        })
      }

      lastDrawnPoint = lastDrawnPoint || new PIXI.Point()
      lastDrawnPoint.set(x, y)

      // 根據畫布大小調整刮除面積計算
      const brushArea = Math.PI * currentBrushSize * currentBrushSize
      scratchedArea.value += brushArea / 10
      scratchProgress.value = scratchedArea.value / (stageSize.width * stageSize.height)

      if (scratchProgress.value >= 0.7) {
        isFinished.value = true
        texture.mask = null
        cover.alpha = 0
      }
    }
  }

  function pointerDown(event) {
    dragging = true
    pointerMove(event)
  }

  function pointerUp(event) {
    dragging = false
    lastDrawnPoint = null
  }
}

onMounted(async () => {
  await nextTick()
  await initGame()

  // 添加視窗大小改變監聽器
  window.addEventListener('resize', handleResize)

  // 使用 ResizeObserver 監聽容器大小變化
  if (root.value && 'ResizeObserver' in window) {
    resizeObserver = new ResizeObserver(handleResize)
    resizeObserver.observe(root.value.parentElement)
  }
})

onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
  if (resizeObserver) {
    resizeObserver.disconnect()
  }
  if (app) {
    app.destroy(true)
  }
})
</script>

<template>
  <div class="scratch-card-container">
    <h1 class="title">🎰 刮刮樂遊戲</h1>
    <div class="game-wrapper">
      <div class="root" :style="{ width: gameSize.width + 'px', height: gameSize.height + 'px' }">
        <div ref="root"></div>
      </div>
    </div>

    <div class="controls">
      <button class="btn restart-btn" @click="restartGame">🔄 重新開始</button>
    </div>
    <div class="result-message" v-if="isFinished">🎉 恭喜！您已刮開獎品！</div>
  </div>
</template>

<style scoped lang="scss">
.progress-bar {
  margin: 10px 0;
}
.progress-bar {
  max-width: 200px;
  height: 14px;
}
.progress-bar {
  max-width: 250px;
  height: 16px;

  .progress-text {
    font-size: 10px;
  }
}
.scratch-card-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
  min-height: 100vh;
  box-sizing: border-box;
}

.title {
  font-size: clamp(1.5rem, 4vw, 2.5rem);
  color: #ffffff;
  margin-bottom: 20px;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.3);
  text-align: center;
  font-weight: bold;
}

.game-wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-bottom: 20px;
  width: 100%;
}

.root {
  margin: 0 auto;
  border-radius: 15px;
  overflow: hidden;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.2);
  transition: all 0.3s ease;

  &:hover {
    transform: translateY(-2px);
    box-shadow: 0 12px 40px rgba(0, 0, 0, 0.4);
  }
}

.controls {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 10px;
  margin: 20px 0;

  .btn {
    background: linear-gradient(45deg, #ff6b6b, #ee5a52);
    color: white;
    border: none;
    padding: 12px 24px;
    font-size: clamp(14px, 2.5vw, 16px);
    border-radius: 25px;
    cursor: pointer;
    box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
    transition: all 0.3s ease;
    font-weight: 500;
    min-width: 120px;

    &:hover {
      transform: translateY(-2px);
      box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3);
    }

    &:active {
      transform: translateY(0);
    }

    &.restart-btn {
      background: linear-gradient(45deg, #4caf50, #45a049);
    }
  }
}

.result-message {
  background: linear-gradient(45deg, #ffd700, #ffa500);
  color: #333;
  padding: 15px 30px;
  border-radius: 25px;
  font-size: clamp(16px, 3vw, 20px);
  font-weight: bold;
  text-align: center;
  box-shadow: 0 4px 20px rgba(255, 215, 0, 0.4);
  animation: bounce 0.6s ease-in-out;
  margin-top: 20px;
}

@keyframes bounce {
  0%,
  20%,
  50%,
  80%,
  100% {
    transform: translateY(0);
  }
  40% {
    transform: translateY(-10px);
  }
  60% {
    transform: translateY(-5px);
  }
}

// 平板電腦
@media screen and (max-width: 1024px) {
  .scratch-card-container {
    padding: 15px;
  }

  .title {
    margin-bottom: 15px;
  }
}

// 手機直向
@media screen and (max-width: 768px) {
  .scratch-card-container {
    padding: 10px;
  }

  .title {
    margin-bottom: 10px;
  }

  .progress-bar {
    max-width: 250px;
    height: 16px;

    .progress-text {
      font-size: 10px;
    }
  }

  .controls .btn {
    padding: 10px 20px;
    min-width: 100px;
  }
}

// 小螢幕手機
@media screen and (max-width: 480px) {
  .scratch-card-container {
    padding: 8px;
  }

  .progress-bar {
    max-width: 200px;
    height: 14px;
  }

  .result-message {
    padding: 12px 20px;
    margin-top: 15px;
  }
}

// 橫向手機
@media screen and (max-height: 500px) and (orientation: landscape) {
  .scratch-card-container {
    padding: 10px;
  }

  .title {
    font-size: 1.5rem;
    margin-bottom: 10px;
  }

  .progress-bar {
    margin: 10px 0;
  }

  .controls {
    margin: 10px 0;
  }
}
</style>
