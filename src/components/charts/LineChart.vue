<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

const props = defineProps({
  // [{ planId, label, date, value }]
  data: { type: Array, default: () => [] },
  max: { type: Number, default: 5 },
  color: { type: String, default: '#4f6ef7' },
  emptyText: { type: String, default: '暂无数据' },
})

const H = 240
const PAD = { top: 24, right: 36, bottom: 44, left: 36 }
const innerH = H - PAD.top - PAD.bottom

// 跟随容器实际宽度布局，文字不按比例缩放，保证始终可读
const wrap = ref(null)
const width = ref(640)
let observer
onMounted(() => {
  observer = new ResizeObserver((entries) => {
    width.value = entries[0].contentRect.width || 640
  })
  observer.observe(wrap.value)
})
onBeforeUnmount(() => observer?.disconnect())

const innerW = computed(() => Math.max(0, width.value - PAD.left - PAD.right))

// 过滤非法评分并换算坐标，评分收敛到 [0, max]
const points = computed(() => {
  const list = props.data.filter((d) => Number.isFinite(Number(d?.value)))
  const n = list.length
  return list.map((d, i) => {
    const value = Math.min(props.max, Math.max(0, Number(d.value)))
    return {
      ...d,
      value,
      x: PAD.left + (n === 1 ? innerW.value / 2 : (i / (n - 1)) * innerW.value),
      y: PAD.top + innerH - (value / props.max) * innerH,
    }
  })
})

// 至少两个点才连成折线，单个数据只画圆点
const linePath = computed(() =>
  points.value.length > 1
    ? points.value.map((p, i) => `${i ? 'L' : 'M'}${p.x},${p.y}`).join(' ')
    : ''
)

// y 轴刻度：0..max 的整数
const ticks = computed(() =>
  Array.from({ length: props.max + 1 }, (_, i) => ({
    value: i,
    y: PAD.top + innerH - (i / props.max) * innerH,
  }))
)

// 数据点较多时隔点显示 x 轴标签，避免重叠
const labelStep = computed(
  () => Math.ceil(points.value.length / Math.max(2, Math.floor(width.value / 90))) || 1
)

function shortLabel(label) {
  const s = String(label ?? '')
  return s.length > 5 ? `${s.slice(0, 5)}…` : s
}

function shortDate(date) {
  return String(date || '').slice(5, 10)
}
</script>

<template>
  <div ref="wrap" class="linechart">
    <svg v-if="points.length" :width="width" :height="H" role="img" aria-label="评分走势折线图">
      <!-- 横向网格线与 y 轴刻度 -->
      <g v-for="t in ticks" :key="t.value">
        <line class="grid" :x1="PAD.left" :x2="width - PAD.right" :y1="t.y" :y2="t.y" />
        <text class="tick" :x="PAD.left - 8" :y="t.y + 4" text-anchor="end">{{ t.value }}</text>
      </g>

      <!-- 评分折线 -->
      <path v-if="linePath" class="line" :d="linePath" :stroke="color" />

      <!-- 数据点与数值 -->
      <g v-for="(p, i) in points" :key="p.planId || i">
        <circle class="dot" :cx="p.x" :cy="p.y" r="4" :fill="color">
          <title>{{ p.label }} · {{ p.date }} · {{ p.value }}分</title>
        </circle>
        <text class="value" :x="p.x" :y="p.y - 10" text-anchor="middle">{{ p.value }}</text>
      </g>

      <!-- x 轴：出行名称 + 出发日期 -->
      <template v-for="(p, i) in points" :key="`x-${p.planId || i}`">
        <g v-if="i % labelStep === 0">
          <text class="x-label" :x="p.x" :y="H - 24" text-anchor="middle">{{ shortLabel(p.label) }}</text>
          <text class="x-date" :x="p.x" :y="H - 10" text-anchor="middle">{{ shortDate(p.date) }}</text>
        </g>
      </template>
    </svg>
    <p v-else class="empty">{{ emptyText }}</p>
  </div>
</template>

<style scoped>
.linechart {
  width: 100%;
}

.linechart svg {
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

.line {
  fill: none;
  stroke-width: 2;
  stroke-linecap: round;
  stroke-linejoin: round;
}

.dot {
  stroke: #fff;
  stroke-width: 2;
}

.value {
  font-size: 11px;
  font-weight: 600;
  fill: var(--text-secondary);
}

.x-label {
  font-size: 11px;
  fill: var(--text-secondary);
}

.x-date {
  font-size: 10px;
  fill: var(--text-muted);
}
</style>
