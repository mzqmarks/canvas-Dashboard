<template>
  <div class="speedometer">
    <div class="canvas-container">
      <canvas ref="canvas"></canvas>
    </div>
    <div class="controls">
      <input type="range" v-model="speed" min="0" max="240" step="1" />
      <span>{{ speed }} km/h</span>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, onMounted, onBeforeUnmount } from 'vue'

const canvas = ref(null)
const speed = ref(0)
const config = ref({
  radius: 0,
  centerX: 0,
  centerY: 0,
  maxSpeed: 240,
  minAngle: -135,
  maxAngle: 135,
})

// 计算仪表盘尺寸
function calculateDimensions() {
  const container = canvas.value.parentElement
  const viewportWidth = window.innerWidth
  const viewportHeight = window.innerHeight

  // 根据视口宽高比选择最小边作为基准
  const isLandscape = viewportWidth > viewportHeight
  const baseSize = isLandscape ? viewportHeight : viewportWidth

  // 设置canvas为正方形
  const size = baseSize * 0.9 // 留出10%的边距
  canvas.value.width = size
  canvas.value.height = size

  // 更新配置
  const padding = 20 // 增加边距
  config.value.radius = (size - padding * 2) * 0.45 // 调整半径比例
  config.value.centerX = size / 2
  config.value.centerY = size / 2

  // 重绘仪表盘
  const ctx = canvas.value.getContext('2d')
  drawSpeedometer(ctx)

  // 调整容器尺寸
  container.style.width = `${size}px`
  container.style.height = `${size}px`
}

// 绘制仪表盘
function drawSpeedometer(ctx) {
  // 完全清空画布
  ctx.clearRect(0, 0, canvas.value.width, canvas.value.height)
  // 重置画布状态
  ctx.setTransform(1, 0, 0, 1, 0, 0)

  // 创建渐变色
  const gradient = ctx.createLinearGradient(
    0,
    0,
    config.value.centerX * 2,
    config.value.centerY * 2
  )
  gradient.addColorStop(0, '#00ff00') // 绿色
  gradient.addColorStop(0.5, '#ffff00') // 黄色
  gradient.addColorStop(1, '#ff0000') // 红色

  // 绘制背景圆环
  ctx.beginPath()
  const ringRadius = config.value.radius - 5 // 缩小5px确保完全显示
  ctx.arc(
    config.value.centerX,
    config.value.centerY,
    ringRadius,
    0,
    2 * Math.PI
  )
  ctx.strokeStyle = '#e0e0e0'
  ctx.lineWidth = 10
  ctx.stroke()

  // 绘制动态渐变区域
  const currentAngle =
    config.value.minAngle +
    (speed.value / config.value.maxSpeed) *
      (config.value.maxAngle - config.value.minAngle)

  // 保存当前画布状态
  ctx.save()

  // 创建剪切区域
  ctx.beginPath()
  ctx.moveTo(config.value.centerX, config.value.centerY)
  ctx.arc(
    config.value.centerX,
    config.value.centerY,
    config.value.radius,
    (config.value.minAngle * Math.PI) / 180,
    (currentAngle * Math.PI) / 180
  )
  ctx.closePath()
  ctx.clip()

  // 绘制渐变圆环
  ctx.beginPath()
  ctx.arc(
    config.value.centerX,
    config.value.centerY,
    config.value.radius,
    0,
    2 * Math.PI
  )
  ctx.strokeStyle = gradient
  ctx.lineWidth = 10
  ctx.stroke()

  // 恢复画布状态
  ctx.restore()

  // 绘制刻度
  const totalTicks = 24
  const angleStep = (config.value.maxAngle - config.value.minAngle) / totalTicks
  ctx.font = '12px Arial'
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  ctx.fillStyle = '#333'

  for (let i = 0; i <= totalTicks; i++) {
    const angle = config.value.minAngle + i * angleStep
    const start = {
      x:
        config.value.centerX +
        (config.value.radius - 20) * Math.cos((angle * Math.PI) / 180),
      y:
        config.value.centerY +
        (config.value.radius - 20) * Math.sin((angle * Math.PI) / 180),
    }
    const end = {
      x:
        config.value.centerX +
        config.value.radius * Math.cos((angle * Math.PI) / 180),
      y:
        config.value.centerY +
        config.value.radius * Math.sin((angle * Math.PI) / 180),
    }

    // 绘制刻度线
    ctx.beginPath()
    ctx.moveTo(start.x, start.y)
    ctx.lineTo(end.x, end.y)
    ctx.strokeStyle = '#666'
    ctx.lineWidth = 2
    ctx.stroke()

    // 每2个刻度显示一个数字
    if (i % 2 === 0) {
      const speedValue = (i / 2) * 20
      const textPos = {
        x:
          config.value.centerX +
          (config.value.radius + 20) * Math.cos((angle * Math.PI) / 180),
        y:
          config.value.centerY +
          (config.value.radius + 20) * Math.sin((angle * Math.PI) / 180),
      }
      ctx.fillText(speedValue.toString(), textPos.x, textPos.y)
    }
  }

  // 绘制指针
  const angle =
    config.value.minAngle +
    (speed.value / config.value.maxSpeed) *
      (config.value.maxAngle - config.value.minAngle)
  const pointerLength = config.value.radius - 30
  const pointerEnd = {
    x: config.value.centerX + pointerLength * Math.cos((angle * Math.PI) / 180),
    y: config.value.centerY + pointerLength * Math.sin((angle * Math.PI) / 180),
  }

  // 保存当前画布状态
  ctx.save()

  // 设置指针样式
  ctx.strokeStyle = '#ff0000'
  ctx.lineWidth = 2
  ctx.lineCap = 'round'

  // 创建指针阴影
  ctx.shadowColor = 'rgba(0,0,0,0.3)'
  ctx.shadowBlur = 5
  ctx.shadowOffsetX = 2
  ctx.shadowOffsetY = 2

  // 绘制渐变宽度指针
  const pointerWidth = 6 // 根部宽度
  const pointerTipWidth = 1 // 顶部宽度

  // 计算指针两侧的控制点
  const angleRad = (angle * Math.PI) / 180
  const perpendicularAngle = angleRad + Math.PI / 2

  const startOffset = {
    x: (Math.cos(perpendicularAngle) * pointerWidth) / 2,
    y: (Math.sin(perpendicularAngle) * pointerWidth) / 2,
  }

  const endOffset = {
    x: (Math.cos(perpendicularAngle) * pointerTipWidth) / 2,
    y: (Math.sin(perpendicularAngle) * pointerTipWidth) / 2,
  }

  // 绘制指针形状
  ctx.beginPath()
  ctx.moveTo(
    config.value.centerX + startOffset.x,
    config.value.centerY + startOffset.y
  )
  ctx.lineTo(pointerEnd.x + endOffset.x, pointerEnd.y + endOffset.y)
  ctx.quadraticCurveTo(
    pointerEnd.x,
    pointerEnd.y,
    pointerEnd.x - endOffset.x,
    pointerEnd.y - endOffset.y
  )
  ctx.lineTo(
    config.value.centerX - startOffset.x,
    config.value.centerY - startOffset.y
  )
  ctx.quadraticCurveTo(
    config.value.centerX,
    config.value.centerY,
    config.value.centerX + startOffset.x,
    config.value.centerY + startOffset.y
  )
  ctx.closePath()

  // 填充指针
  ctx.fillStyle = '#ff0000'
  ctx.fill()

  // 添加圆角效果
  ctx.lineJoin = 'round'
  ctx.lineWidth = 2
  ctx.strokeStyle = '#ff0000'
  ctx.stroke()

  // 恢复画布状态
  ctx.restore()
}

// 初始化画布
function initCanvas() {
  const ctx = canvas.value.getContext('2d')
  drawSpeedometer(ctx)
}

// 监听速度变化
watch(speed, () => {
  const ctx = canvas.value.getContext('2d')
  drawSpeedometer(ctx)
})

onMounted(() => {
  // 立即计算尺寸并绘制
  calculateDimensions()
  window.addEventListener('resize', calculateDimensions)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', calculateDimensions)
})
</script>

<style scoped>
.speedometer {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
  height: 100vh;
  padding: 20px;
}

.canvas-container {
  flex: 1;
  width: 100%;
  height: 100%;
  min-width: 300px;
  min-height: 300px;
  max-width: 800px;
  max-height: 800px;
  aspect-ratio: 1;
  margin: 0 auto;
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
  resize: both;
}

canvas {
  position: absolute;
  width: 100%;
  height: 100%;
  border: 1px solid #ccc;
  border-radius: 50%;
}

.controls {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px;
}

input[type='range'] {
  width: min(100%, 400px);
}
</style>
