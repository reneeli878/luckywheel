<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue'

const props = defineProps({
  spinning: { type: Boolean, default: false },
  celebrating: { type: Boolean, default: false }
})

const products = [
  'cube-pink', 'cube-silver', 'cube-black',
  'cube-blackgold', 'cube-blue', 'cube-gold',
  'duo-black', 'duo-blackgold', 'duo-blue',
  'duo-gold', 'duo-pink', 'duo-silver',
  'mask-black', 'mask-blue', 'mask-grayveil',
  'mask-red', 'mask-wood'
]
const baseUrl = import.meta.env.BASE_URL
const angle = ref(0)
const intensity = ref(0)
const elapsed = ref(0)
let raf = 0
let last = 0
let spinElapsed = 0
let spinStartAngle = 0
const SPIN_SECONDS = 5 // 與 App.vue 的 5000ms 輪盤動畫一致
const SPIN_DEGREES = 1080 // 抽獎期間環繞三圈

// 動畫由 spinning 的狀態觸發；停止時不重新設定角度。
watch(() => props.spinning, now => {
  if (now) {
    spinElapsed = 0
    spinStartAngle = angle.value
  } else {
    // 抽獎結束立刻固定在終點，不再額外轉 4.7 秒。
    angle.value = (spinStartAngle + SPIN_DEGREES) % 360
    intensity.value = 0
  }
})

function frame(now) {
  if (!last) last = now
  const dt = Math.min((now - last) / 1000, 0.05)
  last = now
  if (props.spinning) {
    spinElapsed = Math.min(SPIN_SECONDS, spinElapsed + dt)
    const t = spinElapsed / SPIN_SECONDS
    // 一開始快速旋轉，接近結束時逐漸減速至 0。
    const eased = 1 - Math.pow(1 - t, 3)
    angle.value = (spinStartAngle + SPIN_DEGREES * eased) % 360
    // 隨著速度降低，擺動幅度也一起縮小。
    intensity.value = Math.pow(1 - t, 2)
    elapsed.value += dt
  }
  raf = requestAnimationFrame(frame)
}

onMounted(() => { raf = requestAnimationFrame(frame) })
onUnmounted(() => { cancelAnimationFrame(raf) })

function dancerStyle(index) {
  const a = angle.value + index * 360 / products.length
  const rad = (a - 90) * Math.PI / 180
  // 每台機器錯開節拍，且擺動強度與旋轉速度同步變化。
  const beat = elapsed.value * (3.2 + intensity.value * 8.5) + index * 0.75
  const sway = Math.sin(beat) * (intensity.value * 30)
  const bounce = Math.abs(Math.sin(beat * 0.85)) * (intensity.value * 22)
  const lean = Math.sin(beat * 0.5 + index) * intensity.value * 8
  return {
    left: `${50 + 57 * Math.cos(rad)}%`,
    top: `${50 + 57 * Math.sin(rad)}%`,
    '--sway': `${sway + lean}deg`,
    '--bounce': `${-bounce}px`,
    '--scale': `${1 + intensity.value * 0.12 * Math.abs(Math.sin(beat))}`
  }
}
</script>

<template>
  <div class="orbit-scene" aria-hidden="true">
    <div
      v-for="(name, index) in products"
      :key="name"
      class="orbit-position"
      :style="dancerStyle(index)"
    >
      <div class="orbit-upright">
        <img
          class="orbit-image"
          :src="`${baseUrl}assets/products/${name}.png`"
          :alt="name"
          draggable="false"
          :style="{
            transform: `translateY(var(--bounce)) rotate(var(--sway)) scale(var(--scale))`
          }"
        />
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Positions are calculated on a real circular path, not by rotating full-size layers. */
.orbit-scene {
  position: absolute;
  inset: 0;
  z-index: 3;
  pointer-events: none;
  overflow: visible;
}
.orbit-position {
  position: absolute;
  width: 0;
  height: 0;
  /* No full-size rotating boxes: each dancer is anchored at a point. */
}
.orbit-upright {
  position: absolute;
  width: 44px;
  height: 44px;
  left: -22px;
  top: -22px;
}
.orbit-image {
  display: block;
  width: 44px !important;
  height: 44px !important;
  max-width: none !important;
  object-fit: contain;
  transform-origin: 50% 75%;
  will-change: transform;
}
@media (max-width: 600px) {
  .orbit-upright { width: 30px; height: 30px; left: -15px; top: -15px; }
  .orbit-image { width: 30px !important; height: 30px !important; }
}
@media (prefers-reduced-motion: reduce) {
  .orbit-image { will-change: auto; }
}
</style>
