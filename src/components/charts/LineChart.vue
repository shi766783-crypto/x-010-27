<script setup>
import { computed } from 'vue'

const props = defineProps({
  // [{ label, value, date }]
  data: { type: Array, default: () => [] },
  min: { type: Number, default: 0 },
  max: { type: Number, default: 5 },
  formatter: { type: Function, default: (v) => v },
})

const PAD = { top: 30, right: 28, bottom: 46, left: 36 }
const HEIGHT = 260
const ITEM_W = 72

// 过滤掉缺失/非法的评分，保证图表始终可渲染
const points = computed(() =>
  (props.data || [])
    .map((d) => ({ ...d, value: Number(d.value) }))
    .filter((d) => Number.isFinite(d.value))
)

const width = computed(() =>
  Math.max(480, PAD.left + PAD.right + Math.max(points.value.length - 1, 1) * ITEM_W)
)
const innerW = computed(() => width.value - PAD.left - PAD.right)
const innerH = HEIGHT - PAD.top - PAD.bottom

function x(i) {
  const n = points.value.length
  if (n <= 1) return PAD.left + innerW.value / 2
  return PAD.left + (innerW.value * i) / (n - 1)
}

function y(v) {
  const range = Math.max(1, props.max - props.min)
  const ratio = Math.min(1, Math.max(0, (v - props.min) / range))
  return PAD.top + innerH * (1 - ratio)
}

const coords = computed(() =>
  points.value.map((p, i) => ({ ...p, cx: x(i), cy: y(p.value) }))
)

const linePoints = computed(() => coords.value.map((c) => `${c.cx},${c.cy}`).join(' '))

// 折线下方的浅色面积区域
const areaPoints = computed(() => {
  if (coords.value.length < 2) return ''
  const base = PAD.top + innerH
  return `${coords.value[0].cx},${base} ${linePoints.value} ${coords.value[coords.value.length - 1].cx},${base}`
})

const ticks = computed(() => {
  const span = props.max - props.min
  const step = span <= 6 ? 1 : Math.ceil(span / 6)
  const list = []
  for (let v = props.min; v <= props.max; v += step) list.push(v)
  if (list[list.length - 1] !== props.max) list.push(props.max)
  return list
})

// YYYY-MM-DD -> M/D，无法解析时留空
function shortDate(date) {
  const parts = typeof date === 'string' ? date.split('-') : []
  return parts.length === 3 ? `${Number(parts[1])}/${Number(parts[2])}` : ''
}

function shortLabel(label) {
  const s = String(label ?? '')
  return s.length > 6 ? `${s.slice(0, 6)}…` : s
}
</script>

<template>
  <p v-if="!coords.length" class="empty">还没有出行评分，完成出行总结并评分后即可查看评分走势</p>

  <div v-else class="line-scroll">
    <svg
      class="line-chart"
      :width="width"
      :height="HEIGHT"
      :viewBox="`0 0 ${width} ${HEIGHT}`"
      role="img"
      aria-label="出行评分走势"
    >
      <!-- 横向网格线与 Y 轴刻度 -->
      <g v-for="t in ticks" :key="t">
        <line class="grid" :x1="PAD.left" :x2="width - PAD.right" :y1="y(t)" :y2="y(t)" />
        <text class="tick" :x="PAD.left - 8" :y="y(t) + 4" text-anchor="end">{{ t }}</text>
      </g>

      <!-- 面积与折线 -->
      <polygon v-if="areaPoints" class="area" :points="areaPoints" />
      <polyline v-if="coords.length > 1" class="line" :points="linePoints" />

      <!-- 数据点、数值与 X 轴标签 -->
      <g v-for="(c, i) in coords" :key="i">
        <title>{{ c.label }} {{ shortDate(c.date) }}：{{ formatter(c.value) }}</title>
        <circle class="dot" :cx="c.cx" :cy="c.cy" r="4.5" />
        <text class="value" :x="c.cx" :y="c.cy - 10" text-anchor="middle">{{ formatter(c.value) }}</text>
        <text class="axis" :x="c.cx" :y="HEIGHT - 26" text-anchor="middle">{{ shortLabel(c.label) }}</text>
        <text class="axis date" :x="c.cx" :y="HEIGHT - 12" text-anchor="middle">{{ shortDate(c.date) }}</text>
      </g>
    </svg>
  </div>
</template>

<style scoped>
.line-scroll {
  overflow-x: auto;
}

.line-chart {
  display: block;
}

.grid {
  stroke: #eef0f4;
  stroke-width: 1;
}

.tick {
  font-size: 11px;
  fill: var(--text-muted);
}

.area {
  fill: var(--primary);
  opacity: 0.08;
}

.line {
  fill: none;
  stroke: var(--primary);
  stroke-width: 2.5;
  stroke-linejoin: round;
  stroke-linecap: round;
}

.dot {
  fill: #fff;
  stroke: var(--primary);
  stroke-width: 2.5;
}

.value {
  font-size: 12px;
  font-weight: 600;
  fill: var(--text);
}

.axis {
  font-size: 11px;
  fill: var(--text-secondary);
}

.axis.date {
  font-size: 10px;
  fill: var(--text-muted);
}
</style>
