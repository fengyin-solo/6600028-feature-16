<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch, nextTick } from 'vue'
import { useFluidStore } from '../store/fluid'

const store = useFluidStore()
const canvas = ref<HTMLCanvasElement | null>(null)

const W = 800
const H = 500

const intensityOptions = [
  { value: 'low', label: '低' },
  { value: 'medium', label: '中' },
  { value: 'high', label: '高' },
  { value: 'auto', label: '自动' },
]

const gridSizeOptions = [
  { value: 5, label: '精细' },
  { value: 10, label: '标准' },
  { value: 20, label: '粗糙' },
]

function getIntensityScale(intensity: string, maxDens: number): number {
  switch (intensity) {
    case 'low':
      return maxDens * 0.3
    case 'medium':
      return maxDens * 0.6
    case 'high':
      return maxDens * 1.0
    case 'auto':
    default:
      return maxDens
  }
}

function densityToColor(d: number, scale: number): string {
  const t = Math.min(d / scale, 1)
  const hue = (1 - t) * 240
  const sat = 80
  const light = 30 + t * 40
  return `hsl(${hue}, ${sat}%, ${light}%)`
}

function draw() {
  const ctx = canvas.value?.getContext('2d')
  if (!ctx || !store.engine) return

  ctx.fillStyle = '#0c1222'
  ctx.fillRect(0, 0, W, H)

  const gridSize = store.heatmapGridSize
  const gw = Math.ceil(W / gridSize)
  const gh = Math.ceil(H / gridSize)
  const densityGrid = new Float32Array(gw * gh)

  for (const p of store.engine.particles) {
    const gx = Math.floor(p.x / gridSize)
    const gy = Math.floor(p.y / gridSize)
    if (gx >= 0 && gx < gw && gy >= 0 && gy < gh) {
      densityGrid[gy * gw + gx] += p.density
    }
  }

  const maxDens = Math.max(...densityGrid, 1)
  const scale = getIntensityScale(store.heatmapIntensity, maxDens)

  for (let gy = 0; gy < gh; gy++) {
    for (let gx = 0; gx < gw; gx++) {
      const d = densityGrid[gy * gw + gx]
      if (d > 0) {
        ctx.fillStyle = densityToColor(d, scale)
        ctx.fillRect(gx * gridSize, gy * gridSize, gridSize, gridSize)
      }
    }
  }

  ctx.strokeStyle = '#475569'
  ctx.lineWidth = 3
  ctx.strokeRect(2, 2, W - 4, H - 4)

  drawColorBar(ctx, scale, maxDens)
}

function drawColorBar(ctx: CanvasRenderingContext2D, scale: number, maxDens: number) {
  const barX = W - 40
  const barY = 30
  const barW = 20
  const barH = H - 80
  const steps = 50

  for (let i = 0; i < steps; i++) {
    const t = i / (steps - 1)
    const d = (1 - t) * scale
    ctx.fillStyle = densityToColor(d, scale)
    ctx.fillRect(barX, barY + i * (barH / steps), barW, barH / steps + 1)
  }

  ctx.strokeStyle = '#94a3b8'
  ctx.lineWidth = 1
  ctx.strokeRect(barX, barY, barW, barH)

  ctx.fillStyle = '#e2e8f0'
  ctx.font = '10px monospace'
  ctx.textAlign = 'left'

  const labels = [
    { y: barY, value: scale.toFixed(0) },
    { y: barY + barH / 2, value: (scale / 2).toFixed(0) },
    { y: barY + barH, value: '0' },
  ]

  for (const label of labels) {
    ctx.fillText(label.value, barX - 5, label.y + 3)
  }

  if (store.heatmapIntensity === 'auto') {
    ctx.fillStyle = '#94a3b8'
    ctx.font = '9px sans-serif'
    ctx.fillText(`最大: ${maxDens.toFixed(0)}`, barX - 5, barY - 8)
  }
}

let raf: number | null = null
function animate() {
  draw()
  raf = requestAnimationFrame(animate)
}

function closeViewer() {
  store.toggleHeatmapViewer()
}

watch(() => store.showHeatmapViewer, (val) => {
  if (val) {
    nextTick(() => {
      animate()
    })
  } else {
    if (raf) {
      cancelAnimationFrame(raf)
      raf = null
    }
  }
})

onMounted(() => {
  if (store.showHeatmapViewer) {
    animate()
  }
})

onUnmounted(() => {
  if (raf) cancelAnimationFrame(raf)
})

function selectIntensity(value: string) {
  store.setHeatmapIntensity(value as any)
}

function selectGridSize(value: number) {
  store.setHeatmapGridSize(value)
}
</script>

<template>
  <Teleport to="body">
    <div
      v-if="store.showHeatmapViewer"
      class="fixed inset-0 z-50 flex items-center justify-center bg-black/70 backdrop-blur-sm"
      @click.self="closeViewer"
    >
      <div class="bg-gray-800 rounded-xl border border-gray-700 shadow-2xl max-w-[900px] w-full mx-4">
        <div class="flex items-center justify-between px-5 py-3 border-b border-gray-700">
          <div class="flex items-center gap-3">
            <h2 class="text-lg font-semibold text-white">热力图查看器</h2>
            <span class="text-xs text-gray-500 bg-gray-700/50 px-2 py-0.5 rounded">密度分布</span>
          </div>
          <button
            @click="closeViewer"
            class="text-gray-400 hover:text-white transition p-1 rounded hover:bg-gray-700"
          >
            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <line x1="18" y1="6" x2="6" y2="18"></line>
              <line x1="6" y1="6" x2="18" y2="18"></line>
            </svg>
          </button>
        </div>

        <div class="p-5">
          <div class="flex gap-4 mb-4">
            <div class="flex-1">
              <label class="block text-xs text-gray-400 mb-2">强度层级</label>
              <div class="flex gap-1">
                <button
                  v-for="opt in intensityOptions"
                  :key="opt.value"
                  @click="selectIntensity(opt.value)"
                  class="flex-1 py-1.5 px-2 text-xs rounded transition"
                  :class="store.heatmapIntensity === opt.value
                    ? 'bg-blue-600 text-white'
                    : 'bg-gray-700 text-gray-300 hover:bg-gray-600'"
                >
                  {{ opt.label }}
                </button>
              </div>
            </div>
            <div class="flex-1">
              <label class="block text-xs text-gray-400 mb-2">网格精度</label>
              <div class="flex gap-1">
                <button
                  v-for="opt in gridSizeOptions"
                  :key="opt.value"
                  @click="selectGridSize(opt.value)"
                  class="flex-1 py-1.5 px-2 text-xs rounded transition"
                  :class="store.heatmapGridSize === opt.value
                    ? 'bg-blue-600 text-white'
                    : 'bg-gray-700 text-gray-300 hover:bg-gray-600'"
                >
                  {{ opt.label }}
                </button>
              </div>
            </div>
          </div>

          <div class="relative">
            <canvas
              ref="canvas"
              :width="W"
              :height="H"
              class="rounded-lg border border-gray-600 w-full"
            />
          </div>

          <div class="mt-3 flex items-center justify-between text-xs text-gray-500">
            <div class="flex items-center gap-4">
              <span>粒子数: <span class="text-blue-400 font-mono">{{ store.particleArray.length }}</span></span>
              <span>平均密度: <span class="text-yellow-400 font-mono">{{ store.avgDensity.toFixed(0) }}</span></span>
            </div>
            <div class="flex items-center gap-2">
              <span class="inline-block w-3 h-3 rounded-full" style="background: hsl(240, 80%, 40%)"></span>
              <span>低密度</span>
              <span class="mx-1">→</span>
              <span class="inline-block w-3 h-3 rounded-full" style="background: hsl(0, 80%, 60%)"></span>
              <span>高密度</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </Teleport>
</template>
