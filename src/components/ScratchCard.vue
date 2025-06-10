<script setup>
import { ref, reactive, onMounted } from 'vue'
import * as PIXI from 'pixi.js'

const prizeImages = ['/imgs/prize1.png', '/imgs/prize2.png', '/imgs/prize3.png', '/imgs/prize4.png', '/imgs/prize5.png', '/imgs/prize6.png']

const currentPrize = ref(prizeImages[0]) // 預設第一張

const scratchedArea = ref(0)
const scratchProgress = ref(0)
const isFinished = ref(false)
const root = ref(null)

let app, cover, texture, renderTexture, renderTextureSprite

const restartGame = async () => {
  // 重置響應式變數
  scratchedArea.value = 0
  scratchProgress.value = 0
  isFinished.value = false

  // 重新選擇獎品
  const randomIndex = Math.floor(Math.random() * prizeImages.length)
  currentPrize.value = prizeImages[randomIndex]

  // 重置遊戲狀態
  if (app && cover && texture && renderTexture) {
    // 清空 renderTexture
    app.renderer.render({
      container: new PIXI.Container(),
      target: renderTexture,
      clear: true,
    })

    // 重置 cover 和 texture 狀態
    cover.alpha = 1
    texture.mask = renderTextureSprite
    await PIXI.Assets.load([currentPrize.value])
    // 更新獎品圖片
    texture.texture = PIXI.Texture.from(currentPrize.value)
  }
}

const initGame = async () => {
  const randomIndex = Math.floor(Math.random() * prizeImages.length)
  currentPrize.value = prizeImages[randomIndex]

  // Create a new application
  app = new PIXI.Application()
  // console.log(root)

  await app.init({ resizeTo: document.querySelector('.root'), backgroundColor: 'transparent', margin: 'auto' })

  root.value.appendChild(app.canvas)

  const brush = new PIXI.Graphics().circle(0, 0, 50).fill({ color: 0xffffff })

  const line = new PIXI.Graphics()

  // 用Assets集中載入資源
  await PIXI.Assets.load(['imgs/cover.png', currentPrize.value])

  const { width, height } = app.screen
  const stageSize = { width, height }

  // Sprites can display images, handle input events, and be transformed in various ways.
  cover = Object.assign(PIXI.Sprite.from('imgs/cover.png'), stageSize)
  // 載入獎勵圖片
  texture = Object.assign(PIXI.Sprite.from(currentPrize.value), stageSize)
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
      brush.position.set(x, y)
      app.renderer.render({
        container: brush,
        target: renderTexture,
        clear: false,
        skipUpdateTransform: false,
      })

      if (lastDrawnPoint) {
        line.clear().moveTo(lastDrawnPoint.x, lastDrawnPoint.y).lineTo(x, y).stroke({ width: 100, color: 0xffffff })
        app.renderer.render({
          container: line,
          target: renderTexture,
          clear: false,
          skipUpdateTransform: false,
        })
      }

      lastDrawnPoint = lastDrawnPoint || new PIXI.Point()
      lastDrawnPoint.set(x, y)

      scratchedArea.value += (Math.PI * 50 * 50) / 10
      scratchProgress.value = scratchedArea.value / (600 * 400)
      if (scratchProgress.value >= 0.7) {
        console.log('>0.7')

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
onMounted(initGame)
</script>
<template>
  <h1 class="title">🎰 刮刮樂遊戲</h1>
  <div class="root">
    <div ref="root"></div>
  </div>
  <div class="controls">
    <button class="btn" @click="restartGame">🔄 重新開始</button>
  </div>
</template>

<style scoped lang="scss">
.title {
  font-size: 2.5em;
  color: #ffffff;
  margin-bottom: 20px;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.1);
  text-align: center;
  padding-bottom: 20px;
}
.root {
  margin: 0 auto;
  width: 600px;
  height: 400px;
}

.controls {
  display: flex;
  justify-content: center;
  margin-top: 20px;
  .btn {
    background: linear-gradient(45deg, #ff6b6b, #ee5a52);
    color: white;
    border: none;
    padding: 12px 24px;
    font-size: 16px;
    border-radius: 25px;
    cursor: pointer;
    margin: 5px;
    box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
    transition: all 0.3s ease;
    &:hover {
      transform: translateY(-2px);
      box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3);
    }
    &:active {
      transform: translateY(0);
    }
  }
}

@media screen and (max-width: 768px) {
  .root {
    width: 300px;
    height: 200px;
  }
}
</style>
