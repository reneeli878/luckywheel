<script setup>
const baseUrl = import.meta.env.BASE_URL
import { computed, ref, watch, nextTick, onMounted, onUnmounted } from 'vue'
import OrbitProducts from './components/OrbitProducts.vue'

const defaults = [
  { id: 1, name: '7 折券', weight: 2, stock: 2, color: '#f7b0ad' },
  { id: 2, name: '8 折券', weight: 5, stock: 5, color: '#f6c48e' },
  { id: 3, name: '85 折券', weight: 8, stock: 8, color: '#f5dc90' },
  { id: 4, name: '9 折券', weight: 15, stock: 15, color: '#a9d7c0' },
  { id: 5, name: '95 折券', weight: 20, stock: 20, color: '#99cfe2' },
  { id: 6, name: '98 折券', weight: 35, stock: 35, color: '#bcb6e8' },
  { id: 7, name: '帆布袋', weight: 5, stock: 5, color: '#eab5d5' },
  { id: 8, name: '燈箱', weight: 10, stock: 10, color: '#9fd7d5' },
]
const key = 'luftqi-wheel-v1'
function load() {
  try {
    const saved = JSON.parse(localStorage.getItem(key))
    if (saved && Array.isArray(saved.prizes) && Array.isArray(saved.history)) return saved
  } catch { /* corrupted storage: start fresh */ }
  return { prizes: defaults, history: [] }
}
const saved = load()
// 初始庫存獨立保存，抽獎扣庫存時不會改動。舊版資料依預設獎品回填。
const prizes = ref(saved.prizes.map(p => ({
  ...p,
  initialStock: Number.isInteger(Number(p.initialStock)) && Number(p.initialStock) >= 0
    ? Number(p.initialStock)
    : (defaults.find(d => d.id === p.id && d.name === p.name)?.stock ?? Math.max(0, Math.floor(Number(p.stock) || 0)))
})))
const history = ref(saved.history)
const tab = ref('wheel')
const rotation = ref(0)
const spinning = ref(false)
const wheelTransition = ref('none')
const result = ref('')
const showWinnerModal = ref(false)
const confirmAction = ref(null)
// 抽獎期間鎖住扇形，避免動畫還沒停就重新分配機率
const lockedSegments = ref(null)
const displayedSegments = computed(() => lockedSegments.value ?? segments.value)
const remainingStock = computed(() => prizes.value.reduce((sum, p) => sum + Math.max(0, Math.floor(Number(p.stock) || 0)), 0))

// 統計資料依目前設定與抽獎紀錄即時計算，不另存一份，避免數字不同步。
const totalInitialStock = computed(() => prizes.value.reduce((sum, p) => sum + Math.max(0, Number(p.initialStock) || 0), 0))
const totalWeight = computed(() => remainingStock.value)
const awardedCount = computed(() => history.value.length)
const awardedPrizeKinds = computed(() => new Set(history.value.map(h => String(h.name))).size)
const prizeDrawStats = computed(() => {
  const countsById = new Map()
  const legacyByName = new Map()
  for (const h of history.value) {
    if (h.prizeId != null) countsById.set(String(h.prizeId), (countsById.get(String(h.prizeId)) || 0) + 1)
    else legacyByName.set(String(h.name), (legacyByName.get(String(h.name)) || 0) + 1)
  }
  const rows = prizes.value.map(p => ({
    id: p.id, name: p.name, color: p.color, stock: Math.max(0, Number(p.stock) || 0),
    count: (countsById.get(String(p.id)) || 0) + (legacyByName.get(String(p.name)) || 0)
  }))
  // 被刪除的獎品仍保留在抽獎紀錄中，統計不能憑空消失。
  const knownIds = new Set(prizes.value.map(p => String(p.id)))
  const knownNames = new Set(prizes.value.map(p => String(p.name)))
  for (const h of history.value) {
    const isUnknown = h.prizeId != null ? !knownIds.has(String(h.prizeId)) : !knownNames.has(String(h.name))
    if (!isUnknown) continue
    const key = h.prizeId != null ? `id:${h.prizeId}` : `name:${h.name}`
    let row = rows.find(r => r.deletedKey === key)
    if (!row) { row = { deletedKey:key, id:key, name:h.name, color:'#cbd5e1', stock:null, count:0, removed:true }; rows.push(row) }
    row.count++
  }
  return rows.sort((a,b) => b.count - a.count || String(a.name).localeCompare(String(b.name), 'zh-TW'))
})

const newPrize = ref({ name: '', weight: 10, stock: 10 })
const colors = ['#f7b0ad', '#f6c48e', '#f5dc90', '#a9d7c0', '#99cfe2', '#bcb6e8', '#eab5d5', '#9fd7d5']
watch([prizes, history], () => {
  localStorage.setItem(key, JSON.stringify({ prizes: prizes.value, history: history.value }))
}, { deep: true })

// 固定獎池、不放回：每一份剩餘庫存都是一張等機率的籤。
// 畫面、指針和抽獎使用相同的庫存比例，權重欄位僅保留舊資料相容性。
const active = computed(() => prizes.value.filter(p => Math.floor(Number(p.stock) || 0) > 0)
  .sort((a, b) => Math.floor(Number(b.stock) || 0) - Math.floor(Number(a.stock) || 0)))
const sortedPrizes = computed(() => [...prizes.value].sort((a, b) =>
  Math.floor(Number(b.stock) || 0) - Math.floor(Number(a.stock) || 0)))
const total = computed(() => active.value.reduce((sum, p) => sum + Math.floor(Number(p.stock) || 0), 0))
const segments = computed(() => {
  let start = 0
  return active.value.map(p => {
    const span = Math.floor(Number(p.stock) || 0) / total.value * 360
    const segment = { ...p, start, span, end: start + span }
    start += span
    return segment
  })
})
const wheelGradient = computed(() => displayedSegments.value.length
  ? `conic-gradient(from 0deg, ${displayedSegments.value.map(s => `${s.color} ${s.start}deg ${s.end}deg`).join(', ')})`
  : 'conic-gradient(#e5e7eb 0deg 360deg)')
// 每個有效獎項都顯示文字，包括 3% 的小扇形。
// 小扇形改為沿半徑排列文字，避免文字被弧長不足的條件隱藏。
const showWheelLabel = () => true
const labelStyle = s => {
  const mid = s.start + s.span / 2
  const narrow = s.span < 13
  const radius = narrow ? 37 : 32
  const rad = (mid - 90) * Math.PI / 180
  return {
    left: `${50 + radius * Math.cos(rad)}%`,
    top: `${50 + radius * Math.sin(rad)}%`,
    transform: narrow ? `translate(-50%, -50%) rotate(${mid - 90}deg)` : 'translate(-50%, -50%)',
    fontSize: narrow ? '11px' : undefined,
    maxWidth: narrow ? '30%' : undefined,
    overflow: 'visible',
    textOverflow: 'clip',
  }
}

async function spin() {
  if (spinning.value || !total.value) return
  // 新一輪開始前，才將上一輪的固定輪盤換成最新權重與庫存。
  // 先無動畫地歸零，再啟動旋轉，避免上一輪的結果被瞬間改寫。
  spinning.value = true
  showWinnerModal.value = false
  result.value = ''
  wheelTransition.value = 'none'
  lockedSegments.value = segments.value.map(s => ({ ...s }))
  rotation.value = 0
  await nextTick()
  // 等瀏覽器真正繪製歸零狀態後才開始動畫。
  await new Promise(resolve => requestAnimationFrame(() => requestAnimationFrame(resolve)))

  const random = crypto.getRandomValues(new Uint32Array(1))[0] / 4294967296
  let draw = random * total.value
  let chosen = active.value[active.value.length - 1]
  for (const p of active.value) {
    draw -= Math.floor(Number(p.stock) || 0)
    if (draw < 0) { chosen = p; break }
  }
  const segment = lockedSegments.value.find(s => s.id === chosen.id)
  // CSS conic-gradient 的 0 度位於正上方，指針也位於正上方。
  const margin = Math.min(segment.span * 0.15, 3)
  const randomOffset = crypto.getRandomValues(new Uint32Array(1))[0] / 4294967296
  const inner = segment.span > 2 * margin
    ? margin + randomOffset * (segment.span - 2 * margin)
    : segment.span / 2
  const target = (360 - (segment.start + inner) % 360) % 360
  wheelTransition.value = 'transform 5s cubic-bezier(.12,.78,.12,1)'
  rotation.value = 360 * 6 + target
  window.setTimeout(() => {
    prizes.value = prizes.value.map(p => p.id === chosen.id
      ? { ...p, stock: Math.max(0, Math.floor(Number(p.stock) || 0) - 1) }
      : p)
    history.value.unshift({ name: chosen.name, at: new Date().toLocaleString('zh-TW'), prizeId: chosen.id })
    result.value = chosen.name
    spinning.value = false
    // 保留本輪扇形及最終旋轉角度，指針才會一直指向真正抽中的獎品。
    // 售罄後的新機率於下一輪抽獎開始時才套用。
    showWinnerModal.value = true
  }, 5000)
}
function updateInitialStock(p) {
  const value = Number(p.stock)
  if (Number.isInteger(value) && value >= 0) p.initialStock = value
}
const resetDescriptions = {
  history: '確定要清除所有抽獎紀錄嗎？目前庫存及機率不會改變，這項操作無法復原。',
  stock: '確定要恢復所有獎品的初始庫存嗎？獎品名稱及抽獎紀錄都會保留，抽獎機率會依恢復後的庫存自動計算。',
  all: '確定要恢復 LUFTQI 預設獎品、機率與庫存，並清空所有抽獎紀錄嗎？自訂獎品也會被移除，這項操作無法復原。'
}
function requestReset(action) {
  if (spinning.value) return
  confirmAction.value = action
}
function executeReset() {
  const action = confirmAction.value
  if (!action || spinning.value) return
  if (action === 'history') history.value = []
  if (action === 'stock') prizes.value = prizes.value.map(p => ({ ...p, stock: Math.max(0, Math.floor(Number(p.initialStock) || 0)) }))
  if (action === 'all') {
    prizes.value = defaults.map(p => ({ ...p, initialStock: p.stock }))
    history.value = []
  }
  result.value = ''
  showWinnerModal.value = false
  lockedSegments.value = null
  rotation.value = 0
  confirmAction.value = null
}
function addPrize() {
  const name = newPrize.value.name.trim()
  const stock = Number(newPrize.value.stock)
  if (!name || !Number.isInteger(stock) || stock < 0) return
  prizes.value.push({ id: Date.now(), name, weight: stock, stock, initialStock: stock, color: colors[prizes.value.length % colors.length] })
  newPrize.value = { name: '', weight: 10, stock: 10 }
}
function downloadHistory() {
  const rows = [['時間', '獎品'], ...history.value.map(h => [h.at, h.name])]
  const csv = '\uFEFF' + rows.map(row => row.map(v => `"${String(v).replaceAll('"', '""')}"`).join(',')).join('\r\n')
  const url = URL.createObjectURL(new Blob([csv], { type: 'text/csv;charset=utf-8' }))
  const link = document.createElement('a')
  link.href = url
  link.download = 'LUFTQI-抽獎紀錄.csv'
  link.click()
  URL.revokeObjectURL(url)
}

// 將 Windows .ani 的兩格動畫轉為 GIF，使用跟隨滑鼠的圖片呈現。
// 不改變原本輪盤指針（紅色三角形）與抽獎機制。
const customCursor = ref({ x: 0, y: 0, visible: false })
const cursorEnabled = ref(false)
let cursorMediaQuery
function moveCustomCursor(event) {
  customCursor.value = { x: event.clientX, y: event.clientY, visible: true }
}
function hideCustomCursor() {
  customCursor.value = { ...customCursor.value, visible: false }
}
function updateCursorMode() {
  cursorEnabled.value = Boolean(cursorMediaQuery?.matches)
  document.documentElement.classList.toggle('luftqi-custom-cursor', cursorEnabled.value)
  if (!cursorEnabled.value) hideCustomCursor()
}
onMounted(() => {
  cursorMediaQuery = window.matchMedia('(hover: hover) and (pointer: fine)')
  updateCursorMode()
  cursorMediaQuery.addEventListener?.('change', updateCursorMode)
  window.addEventListener('pointermove', moveCustomCursor, { passive: true })
  document.addEventListener('pointerleave', hideCustomCursor)
  window.addEventListener('blur', hideCustomCursor)
})
onUnmounted(() => {
  document.documentElement.classList.remove('luftqi-custom-cursor')
  cursorMediaQuery?.removeEventListener?.('change', updateCursorMode)
  window.removeEventListener('pointermove', moveCustomCursor)
  document.removeEventListener('pointerleave', hideCustomCursor)
  window.removeEventListener('blur', hideCustomCursor)
})
</script>

<template>
  <main class="app">
    <Teleport to="body">
      <img v-if="cursorEnabled && customCursor.visible"
        class="luftqi-mouse-cursor"
        :src="`${baseUrl}assets/star-glove-cursor.gif`"
        :style="{ left: `${customCursor.x}px`, top: `${customCursor.y}px` }"
        alt="" aria-hidden="true" draggable="false" />
    </Teleport>
    <header class="topbar">
      <div class="brand-heading"><img class="header-logo" :src="`${baseUrl}assets/luftqi-logo.png`" alt="LUFTQI" /><h1>幸運輪盤 <span>✦</span></h1><p class="subtitle">轉動好運，抽出今日驚喜</p></div>
      <nav><button :class="{ selected: tab === 'wheel' }" @click="tab = 'wheel'">幸運輪盤</button><button :class="{ selected: tab === 'admin' }" @click="tab = 'admin'">獎品設定</button><button :class="{ selected: tab === 'history' }" @click="tab = 'history'">抽獎紀錄</button></nav>
    </header>

    <Transition name="page-swap" mode="out-in" appear>
    <section v-if="tab === 'wheel'" key="wheel" class="wheel-panel">
      <div class="wheel-layout">
        <div class="wheel-stage" :class="{ 'is-spinning': spinning }">
          <OrbitProducts :spinning="spinning" :celebrating="false" />
          <div class="pointer"></div>
          <div class="wheel" :style="{ background: wheelGradient, transform: `rotate(${rotation}deg)`, transition: wheelTransition }">
            <div v-for="s in displayedSegments.filter(showWheelLabel)" :key="s.id" class="wheel-label" :style="labelStyle(s)">{{ s.name }}</div>
          </div>
          <!-- 正式品牌 LOGO 固定於輪盤中央，不跟著旋轉 -->
          <div class="hub"><img :src="`${baseUrl}assets/luftqi-logo.png`" alt="LUFTQI LOGO" /></div>
        </div>
        <aside class="wheel-sidebar">
          <h2 class="legend-title">獎品與中獎機率</h2>
          <div class="prize-legend" aria-label="獎品及中獎機率">
            <div v-for="s in displayedSegments" :key="s.id" class="legend-item">
              <span class="legend-dot" :style="{ backgroundColor: s.color }"></span>
              <span class="legend-name">{{ s.name }}</span>
              <strong>{{ (s.span / 360 * 100).toFixed(1) }}%</strong>
            </div>
          </div>
          <div class="action-area"><p v-if="result" class="winner">🎉 恭喜抽中：{{ result }}</p><p v-else class="hint">{{ spinning ? '輪盤轉動中…' : '準備好試試手氣了嗎？' }}</p><button class="spin-button" :class="{ 'is-spinning': spinning }" :disabled="spinning || !total" @click="spin">{{ spinning ? '抽獎進行中…' : total ? '✦ 開始抽獎' : '獎品已抽完' }}</button><p class="meta">已抽 {{ history.length }} 次 · 可抽獎項 {{ active.length }} 種 · 剩餘 {{ remainingStock }} 份</p></div>
        </aside>
      </div>
    </section>

    <section v-else-if="tab === 'admin'" key="admin" class="panel">
      <h2>獎品與獎池設定</h2><p class="note">固定獎池、不放回：每份獎品都是一張籤。中獎率＝該獎品剩餘庫存 ÷ 全部剩餘庫存，每抽一次即時更新輪盤與機率。依剩餘庫存由多到少排序。</p><p class="note">正式活動建議設定總庫存 100 份。既有抽獎紀錄及剩餘庫存會保留，不會重新補滿。</p>
      <div class="stat-grid" aria-label="獎品設定統計">
        <div class="stat-card"><span>獎品種類</span><strong>{{ prizes.length }}</strong><small>種獎項</small></div>
        <div class="stat-card"><span>目前獎池</span><strong>{{ totalWeight.toLocaleString() }}</strong><small>張可抽的籤</small></div>
        <div class="stat-card"><span>初始總庫存</span><strong>{{ totalInitialStock.toLocaleString() }}</strong><small>份獎品</small></div>
        <div class="stat-card"><span>目前剩餘庫存</span><strong>{{ remainingStock.toLocaleString() }}</strong><small>份可供抽出</small></div>
      </div>
      <div v-for="p in sortedPrizes" :key="p.id" class="prize-row">
        <input type="color" v-model="p.color" aria-label="扇形顏色" /><input v-model="p.name" aria-label="獎品名稱" placeholder="獎品名稱" />
        <label>庫存 <input v-model.number="p.stock" @change="updateInitialStock(p)" type="number" min="0" step="1" /></label>
        <strong>{{ total && p.stock > 0 ? (p.stock / total * 100).toFixed(1) : '0.0' }}%</strong><button class="remove" @click="prizes = prizes.filter(x => x.id !== p.id)">刪除</button>
      </div>
      <div class="new-prize"><h3>新增獎品</h3><input v-model="newPrize.name" placeholder="獎品名稱" /><input v-model.number="newPrize.stock" type="number" min="0" placeholder="庫存" /><button @click="addPrize">＋ 新增獎品</button></div>
    </section>

    <section v-else key="history" class="panel">
      <div class="history-head"><h2>抽獎紀錄</h2><button @click="downloadHistory" :disabled="!history.length">匯出 CSV</button></div>
      <div class="stat-grid" aria-label="抽獎紀錄統計">
        <div class="stat-card"><span>累計抽獎次數</span><strong>{{ awardedCount.toLocaleString() }}</strong><small>次</small></div>
        <div class="stat-card"><span>已抽出獎品種類</span><strong>{{ awardedPrizeKinds }}</strong><small>種</small></div>
        <div class="stat-card"><span>目前剩餘庫存</span><strong>{{ remainingStock.toLocaleString() }}</strong><small>份</small></div>
      </div>
      <div class="draw-summary">
        <h3>各獎品抽出統計</h3>
        <p class="note">統計依下方抽獎紀錄計算。清除紀錄後，抽出次數也會歸零。</p>
        <div class="draw-summary-head"><span>獎品</span><span>抽出次數</span><span>剩餘庫存</span></div>
        <div v-for="item in prizeDrawStats" :key="item.id" class="draw-summary-row">
          <span class="draw-summary-name"><i class="legend-dot" :style="{ backgroundColor:item.color }"></i>{{ item.name }}<small v-if="item.removed">（已刪除）</small></span>
          <strong>{{ item.count }} 次</strong>
          <span>{{ item.stock === null ? '－' : item.stock + ' 份' }}</span>
        </div>
      </div>
      <h3 class="history-detail-title">詳細抽獎紀錄</h3>
      <p v-if="!history.length" class="note">目前尚無抽獎紀錄。</p>
      <div v-for="(h, i) in history" :key="i" class="history-row"><span>{{ h.name }}</span><time>{{ h.at }}</time></div>
      <div class="data-management">
        <h3>資料管理</h3>
        <p class="note">測試結束後可在此清理資料。重置庫存會恢復各獎品設定時的初始數量，機率依剩餘庫存自動計算。</p>
        <div class="data-actions">
          <button type="button" :disabled="spinning || !history.length" @click="requestReset('history')">清除抽獎紀錄</button>
          <button type="button" :disabled="spinning" @click="requestReset('stock')">重置庫存</button>
          <button type="button" class="danger" :disabled="spinning" @click="requestReset('all')">恢復預設設定</button>
        </div>
      </div>
    </section>
    </Transition>
    <Teleport to="body">
      <div v-if="confirmAction" class="reset-overlay" @click.self="confirmAction = null">
        <div class="reset-card" role="alertdialog" aria-modal="true" aria-labelledby="reset-title" aria-describedby="reset-description">
          <h2 id="reset-title">確認資料操作</h2>
          <p id="reset-description">{{ resetDescriptions[confirmAction] }}</p>
          <div class="reset-controls">
            <button type="button" @click="confirmAction = null">取消</button>
            <button type="button" class="confirm-reset" @click="executeReset">確定執行</button>
          </div>
        </div>
      </div>
    </Teleport>
    <Teleport to="body">
      <div v-if="showWinnerModal" class="winner-overlay" @click.self="showWinnerModal = false">
        <div class="celebration-burst" aria-hidden="true">
          <span v-for="i in 40" :key="i" class="celebration-piece" :style="{
            '--x': `${(i * 37 + 11) % 100}%`,
            '--drift': `${((i * 23) % 180) - 90}px`,
            '--delay': `${(i % 9) * 0.065}s`,
            '--duration': `${1.25 + (i % 6) * 0.16}s`,
            '--hue': `${(i * 59) % 360}`,
            '--twist': `${(i % 2 ? 1 : -1) * (240 + (i % 5) * 90)}deg`
          }"></span>
        </div>
        <div class="winner-card" role="dialog" aria-modal="true" aria-labelledby="winner-title">
          <button class="winner-close" type="button" aria-label="關閉中獎畫面" @click="showWinnerModal = false">×</button>
          <img class="winner-brand" :src="`${baseUrl}assets/luftqi-logo.png`" alt="LUFTQI" />
          <div class="winner-confetti" aria-hidden="true">✦ ✨ ✦</div>
          <h2 id="winner-title">🎉 恭喜中獎！</h2>
          <div class="winner-gift" aria-hidden="true">🎁</div>
          <p class="winner-prize">{{ result }}</p>
          <p class="winner-message">幸運降臨，恭喜獲得今日好禮！</p>
          <button class="winner-confirm" type="button" @click="showWinnerModal = false"><span class="winner-confirm-shine" aria-hidden="true"></span><span class="winner-confirm-label">開心收下獎品 ✨</span></button>
        </div>
      </div>
    </Teleport>
    <footer>LUFTQI · 個人化空氣守護</footer>
  </main>
</template>

<style scoped>
.app{min-height:100vh;background:radial-gradient(circle at 50% 20%,#eef7fb 0%,#fff 65%);color:#263a46;padding:28px 22px;font-family:Inter,'Noto Sans TC',system-ui,sans-serif}.topbar{max-width:1100px;margin:auto;display:flex;align-items:center;justify-content:space-between;gap:24px;flex-wrap:wrap}.eyebrow{font-size:12px;letter-spacing:.22em;color:#5c96ae;font-weight:800;margin:0}h1{font-size:36px;margin:6px 0}h1 span{color:#7ab6ce}.subtitle{color:#7c8e9a;margin:0}nav{display:flex;gap:8px;flex-wrap:wrap}button{font:inherit;cursor:pointer}nav button{border:1px solid #dce9ee;border-radius:12px;background:#fff;padding:12px 16px;color:#526d7b}nav button.selected{background:#2d7899;color:#fff;border-color:#2d7899}.wheel-panel{display:flex;align-items:center;flex-direction:column;padding:35px 0}.wheel-stage{position:relative;width:min(76vw,510px);aspect-ratio:1;display:grid;place-items:center}.wheel{width:94%;height:94%;border-radius:50%;border:12px solid #fff;outline:5px solid #a6cddc;box-shadow:0 16px 45px #8fb4c055;position:relative;will-change:transform}.wheel-label{max-width:28%;overflow:hidden;text-overflow:ellipsis;position:absolute;transform:translate(-50%,-50%);font-size:clamp(10px,1.8vw,16px);font-weight:800;color:#34434a;text-shadow:0 1px #ffffffa6;white-space:nowrap;pointer-events:none}.prize-legend{width:min(100%,700px);display:grid;grid-template-columns:repeat(auto-fit,minmax(140px,1fr));gap:9px;margin-top:25px}.legend-item{display:flex;align-items:center;gap:7px;background:#ffffffd9;border:1px solid #e4edf2;border-radius:10px;padding:9px 11px;font-size:13px}.legend-dot{width:12px;height:12px;flex:none;border-radius:50%}.legend-name{flex:1;min-width:0;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}.legend-item strong{color:#2d7899;font-variant-numeric:tabular-nums}.hub{position:absolute;z-index:5;top:50%;left:50%;transform:translate(-50%,-50%);height:76px;width:76px;border-radius:50%;background:white;border:6px solid #c7e1ea;display:grid;place-items:center;font-size:12px;font-weight:900;letter-spacing:.05em;color:#2d7899;box-shadow:0 2px 10px #4562}.pointer{position:absolute;z-index:2;top:-3px;left:50%;transform:translateX(-50%);width:0;height:0;border-left:20px solid transparent;border-right:20px solid transparent;border-top:42px solid #e86e72;filter:drop-shadow(0 3px 2px #7775)}.action-area{text-align:center;margin-top:18px}.hint{color:#6e8795}.winner{font-size:24px;font-weight:900;color:#d45f71}.spin-button{background:#2d7899;color:#fff;border:0;border-radius:50px;font-size:22px;font-weight:800;padding:18px 65px;box-shadow:0 10px 25px #2d789955}.spin-button:disabled{opacity:.6;cursor:not-allowed}.meta{color:#81949f;font-size:13px}.panel{max-width:1000px;margin:42px auto;background:white;border:1px solid #e4edf2;border-radius:22px;padding:26px;box-shadow:0 10px 35px #aac3d01a}.panel h2{margin-top:0}.note{color:#718895;line-height:1.7}.prize-row{display:flex;align-items:center;gap:12px;padding:12px 0;border-bottom:1px solid #edf1f4;flex-wrap:wrap}.prize-row>input:not([type=color]){flex:1;min-width:130px}.prize-row input[type=color]{width:38px;height:38px;padding:2px}.prize-row label{font-size:13px;color:#607987}.prize-row label input{display:block;width:90px;margin-top:4px}.prize-row input,.new-prize input{border:1px solid #d6e3ea;border-radius:9px;padding:9px}.prize-row strong{min-width:56px}.remove{background:#fff1f1;border:0;color:#b34c57;border-radius:8px;padding:10px}.new-prize{display:flex;gap:10px;align-items:center;flex-wrap:wrap;margin-top:24px}.new-prize h3{width:100%;margin:0}.new-prize button,.history-head button{background:#2d7899;color:white;border:0;border-radius:10px;padding:12px 16px}.history-head{display:flex;justify-content:space-between;align-items:center}.history-row{display:flex;justify-content:space-between;border-bottom:1px solid #e9f0f4;padding:13px 0;gap:20px}.history-row time{color:#7b919e}footer{text-align:center;color:#a1b1b9;font-size:12px;margin:25px 0}@media(max-width:600px){.app{padding:20px 12px}h1{font-size:28px}.topbar{justify-content:center;text-align:center}.wheel-stage{width:min(92vw,430px)}.wheel{border-width:7px}.wheel-label{font-size:10px}.hub{width:58px;height:58px;font-size:9px}.spin-button{font-size:18px}.panel{padding:16px}}

/* 左上角品牌圖：使用正式 LOGO，保持原始比例 */
.header-logo { display:block; width:110px; height:auto; object-fit:contain; margin-bottom:8px; }
@media(max-width:600px) { .header-logo { width:96px; margin-inline:auto; } }

/* 正式展覽版：輪盤在左，獎品與操作在右 */
.wheel-panel { display:block; padding:35px 0 20px; }
.wheel-layout { width:min(100%, 1120px); margin:0 auto; display:grid; grid-template-columns:minmax(0, 1fr) 300px; gap:clamp(35px, 5vw, 90px); align-items:center; }
.wheel-layout .wheel-stage { width:min(100%, 510px); justify-self:center; }
.wheel-sidebar { min-width:0; width:100%; }
.legend-title { color:#2d7899; font-size:19px; margin:0 0 15px; }
.wheel-sidebar .prize-legend { width:100%; margin:0; display:grid; grid-template-columns:1fr; gap:9px; }
.wheel-sidebar .legend-item { padding:11px 12px; }
.wheel-sidebar .action-area { margin-top:22px; }
.wheel-sidebar .spin-button { width:100%; padding:16px 12px; font-size:19px; }
/* LOGO 使用橫式原圖，保持比例，不拉伸 */
.wheel-stage .hub { width:74px; height:74px; padding:8px; border-radius:50%; overflow:hidden; }
.wheel-stage .hub img { display:block; width:100%; max-width:100%; height:auto; object-fit:contain; }
@media(max-width:850px) {
  .wheel-layout { grid-template-columns:1fr; gap:55px; }
  .wheel-layout .wheel-stage { width:min(78vw, 470px); }
  .wheel-sidebar { width:min(100%, 600px); justify-self:center; }
  .wheel-sidebar .prize-legend { grid-template-columns:repeat(2, minmax(0,1fr)); }
}
@media(max-width:520px) {
  .wheel-layout { gap:45px; }
  .wheel-layout .wheel-stage { width:min(85vw, 400px); }
  .wheel-stage .hub { width:58px; height:58px; padding:6px; }
  .wheel-sidebar .legend-item { padding:9px 8px; font-size:12px; }
  .wheel-sidebar .spin-button { max-width:360px; }
}

/* 中獎結果：中央放大一次，不循環跳動，也不產生水平捲軸 */
.winner-overlay { position:fixed; inset:0; z-index:9999; box-sizing:border-box; display:flex; justify-content:center; align-items:center; padding:16px; overflow:hidden; background:rgba(20,43,58,.7); backdrop-filter:blur(6px); }
.winner-card { box-sizing:border-box; position:relative; width:min(100%,490px); max-height:calc(100dvh - 32px); overflow-x:hidden; overflow-y:auto; padding:34px 26px 30px; text-align:center; border:2px solid #a6cddc; border-radius:28px; background:linear-gradient(145deg,#fff,#eaf7fc); box-shadow:0 20px 70px #0004; animation:winner-pop .5s ease-out both; }
.winner-close { position:absolute; top:12px; right:16px; border:0; background:transparent; color:#718895; font-size:30px; line-height:1; }
.winner-brand { display:block; width:105px; max-width:45%; height:auto; margin:0 auto 12px; }
.winner-confetti { font-size:24px; line-height:1.3; }
.winner-card h2 { margin:14px 0 8px; color:#2d7899; font-size:clamp(26px,5vw,39px); }
.winner-gift { font-size:76px; line-height:1.5; animation:none; }
.winner-prize { margin:8px 0; color:#d45f71; font-size:clamp(30px,6vw,48px); font-weight:900; overflow-wrap:anywhere; }
.winner-message { color:#718895; font-size:15px; }
.winner-confirm { max-width:100%; margin-top:18px; padding:14px 32px; border:0; border-radius:50px; color:#fff; background:#2d7899; font-size:17px; font-weight:800; }
@keyframes winner-pop { from { opacity:0; transform:scale(.86); } to { opacity:1; transform:scale(1); } }
@media (prefers-reduced-motion:reduce) { .winner-card { animation:none; } }
</style>

<style scoped>
.data-management{margin-top:32px;padding-top:22px;border-top:1px solid #e4edf2}.data-management h3{margin:0 0 6px}.data-actions{display:flex;flex-wrap:wrap;gap:10px;margin-top:14px}.data-actions button{padding:11px 16px;border:1px solid #bfd5df;border-radius:10px;background:#f2f8fa;color:#27627b;font-weight:700}.data-actions button.danger{background:#fff1f1;border-color:#f0c3c5;color:#b34c57}.data-actions button:disabled{opacity:.45;cursor:not-allowed}.reset-overlay{position:fixed;inset:0;z-index:9999;background:#1b2d3b9c;display:grid;place-items:center;padding:20px}.reset-card{background:#fff;border-radius:18px;padding:26px;width:min(100%,420px);box-shadow:0 16px 60px #0003;color:#263a46}.reset-card h2{margin:0 0 12px}.reset-card p{line-height:1.7;color:#607987}.reset-controls{display:flex;justify-content:flex-end;gap:10px;margin-top:20px}.reset-controls button{padding:11px 18px;border-radius:9px;border:1px solid #d4e2e8;background:white}.reset-controls .confirm-reset{background:#b34c57;color:white;border-color:#b34c57}
</style>

<style scoped>
.stat-grid{display:grid;grid-template-columns:repeat(4,minmax(0,1fr));gap:12px;margin:22px 0 26px}
.stat-card{min-width:0;background:linear-gradient(150deg,#f6fbfd,#edf7fb);border:1px solid #dbeaf0;border-radius:14px;padding:15px 16px;display:flex;flex-direction:column;gap:4px}
.stat-card>span{color:#617d8c;font-size:13px}.stat-card strong{color:#246f8e;font-size:29px;line-height:1.35;font-variant-numeric:tabular-nums}.stat-card small{color:#8a9ba5;font-size:12px}
.draw-summary{margin:26px 0 30px}.draw-summary h3{margin-bottom:4px}.draw-summary .note{margin:4px 0 14px;font-size:13px}
.draw-summary-head,.draw-summary-row{display:grid;grid-template-columns:minmax(0,1fr) 120px 120px;align-items:center;gap:12px;padding:12px 15px}
.draw-summary-head{background:#edf5f8;border-radius:10px 10px 0 0;font-size:12px;font-weight:700;color:#637f8e}
.draw-summary-head>:not(:first-child),.draw-summary-row>:not(:first-child){text-align:right}
.draw-summary-row{border-bottom:1px solid #e9f0f4;font-size:14px}.draw-summary-row strong{color:#276f8d;font-variant-numeric:tabular-nums}
.draw-summary-name{display:flex;align-items:center;gap:9px;min-width:0;overflow-wrap:anywhere}.draw-summary-name .legend-dot{display:inline-block}.draw-summary-name small{color:#9aa6ae;font-size:11px}
.history-detail-title{margin:25px 0 8px}
@media(max-width:700px){.stat-grid{grid-template-columns:repeat(2,minmax(0,1fr))}.draw-summary-head,.draw-summary-row{grid-template-columns:minmax(0,1fr) 85px 82px;gap:5px;padding:11px 6px;font-size:12px}.stat-card{padding:12px}.stat-card strong{font-size:25px}}
</style>

<style>
/* 游標需作用於 Teleport、按鈕及輸入欄位，因此使用全域選擇器。 */
@media (hover: hover) and (pointer: fine) {
  html.luftqi-custom-cursor,
  html.luftqi-custom-cursor *,
  html.luftqi-custom-cursor *::before,
  html.luftqi-custom-cursor *::after { cursor: none !important; }
}
.luftqi-mouse-cursor {
  position: fixed;
  z-index: 2147483647;
  width: 32px;
  height: 32px;
  pointer-events: none;
  user-select: none;
  image-rendering: pixelated;
  transform: translate(0, 0);
}
</style>

<style scoped>

/* 微互動：不干擾輪盤旋轉角度或抽獎邏輯 */
.page-swap-enter-active { transition: opacity .30s ease, transform .30s cubic-bezier(.2,.8,.2,1); }
.page-swap-leave-active { transition: opacity .16s ease, transform .16s ease; }
.page-swap-enter-from { opacity:0; transform:translateY(13px) scale(.987); }
.page-swap-leave-to { opacity:0; transform:translateY(-7px); }
.topbar nav button, .app button:not(:disabled) { transition: background-color .22s ease, border-color .22s ease, color .22s ease, box-shadow .22s ease, filter .22s ease, transform .18s cubic-bezier(.2,.8,.2,1); }
.topbar nav button:hover:not(:disabled) { transform:translateY(-3px); box-shadow:0 7px 18px #2d789922; }
.topbar nav button.selected { box-shadow:0 4px 12px #2d78992a; }
.app button:hover:not(:disabled) { filter:brightness(1.055); }
.app button:active:not(:disabled) { transform:translateY(1px) scale(.96); filter:brightness(.97); }
.app button:focus-visible, .app input:focus-visible { outline:3px solid #76bedb; outline-offset:3px; }
.spin-button:hover:not(:disabled) { transform:translateY(-4px) scale(1.025); box-shadow:0 16px 34px #2d789966; }
.spin-button:active:not(:disabled) { transform:translateY(1px) scale(.97); }
.legend-item { transition:transform .22s ease,box-shadow .22s ease,border-color .22s ease; }
.legend-item:hover { transform:translateX(4px); border-color:#b5dce9; box-shadow:0 4px 12px #2d789911; }
.prize-row { transition:background-color .2s ease; border-radius:9px; }
.prize-row:hover { background:#f5fafc; }
.stat-card { transition:transform .22s ease,box-shadow .22s ease; }
.stat-card:hover { transform:translateY(-3px); box-shadow:0 7px 18px #2d789915; }
.draw-summary-row, .history-row { transition:background-color .2s ease; }
.draw-summary-row:hover, .history-row:hover { background:#f3f9fb; }
.reset-card { animation: dialog-arrive .28s cubic-bezier(.18,.85,.2,1.15) both; }
@keyframes dialog-arrive { from { opacity:0;transform:translateY(14px) scale(.96); } to { opacity:1;transform:translateY(0) scale(1); } }
@media (prefers-reduced-motion:reduce) {
 .page-swap-enter-active,.page-swap-leave-active,.topbar nav button,.app button,.legend-item,.prize-row,.stat-card,.draw-summary-row,.history-row { transition:none!important; }
 .reset-card { animation:none!important; }
}

/* 抽獎主按鈕：滑過時掃光，按下時有觸感，旋轉期間持續柔和呼吸 */
.spin-button { position:relative; isolation:isolate; overflow:hidden; transition:transform .2s ease,box-shadow .25s ease,filter .2s ease; }
.spin-button::before { content:""; position:absolute; z-index:-1; inset:0; background:linear-gradient(110deg,transparent 22%,#ffffff60 46%,transparent 67%); transform:translateX(-135%); pointer-events:none; }
.spin-button:hover:not(:disabled)::before { animation:spin-button-sweep .75s ease-out both; }
.spin-button.is-spinning { opacity:1; cursor:wait; animation:spin-button-breathe 1.5s ease-in-out infinite; }
.spin-button.is-spinning::before { animation:spin-button-sweep 1.8s ease-in-out infinite; }
@keyframes spin-button-sweep { from { transform:translateX(-135%); } to { transform:translateX(135%); } }
@keyframes spin-button-breathe { 0%,100% { box-shadow:0 8px 24px #2d789944,0 0 0 0 #6ecdf500; } 50% { box-shadow:0 12px 35px #2d789977,0 0 0 7px #6ecdf52a; } }

/* 輪盤外框的藍白光環：不影響輪盤旋轉及指針判定 */
.wheel-stage::after { content:""; position:absolute; inset:2.2%; z-index:3; border-radius:50%; border:2px solid #bde9f7aa; box-shadow:0 0 13px 3px #9fe1f344, inset 0 0 12px #ffffff99; pointer-events:none; animation:wheel-idle-glow 3.4s ease-in-out infinite; }
.wheel-stage.is-spinning::after { animation:wheel-active-glow 1.15s ease-in-out infinite; }
.wheel-stage .wheel { box-shadow:0 16px 45px #8fb4c055,0 0 23px #8cd4ef44; }
.wheel-stage.is-spinning .pointer { filter:drop-shadow(0 3px 2px #7775) drop-shadow(0 0 9px #ff9e9e); }
@keyframes wheel-idle-glow { 0%,100% { opacity:.46; box-shadow:0 0 10px #91d9ef33; } 50% { opacity:.9; box-shadow:0 0 22px #91d9ef77; } }
@keyframes wheel-active-glow { 0%,100% { opacity:.62; box-shadow:0 0 14px 4px #7bd7f466; } 50% { opacity:1; box-shadow:0 0 29px 8px #7bd7f4aa; } }

/* 中獎彩帶：只在彈窗打開時短暫飄落，完成後自動消失 */
.celebration-burst { position:absolute; inset:0; z-index:1; overflow:hidden; pointer-events:none; }
.celebration-piece { position:absolute; top:-18px; left:var(--x); width:9px; height:15px; border-radius:2px; background:hsl(var(--hue) 83% 67%); opacity:0; animation:confetti-drop var(--duration) cubic-bezier(.2,.65,.45,1) var(--delay) both; }
.celebration-piece:nth-child(3n) { width:7px; height:7px; border-radius:50%; }
.celebration-piece:nth-child(5n) { width:13px; height:6px; }
@keyframes confetti-drop { 0% { opacity:0; transform:translate3d(0,-12px,0) rotate(0deg); } 8% { opacity:1; } 75% { opacity:1; } 100% { opacity:0; transform:translate3d(var(--drift),105vh,0) rotate(var(--twist)); } }
.winner-card { z-index:2; }
/* 彈窗確認按鈕：明顯的浮起、柔和光暈、掃光及按壓效果 */
.winner-confirm { position:relative; isolation:isolate; overflow:hidden; min-height:50px; cursor:pointer; box-shadow:0 7px 20px #2d789944; transition:transform .2s cubic-bezier(.2,.8,.2,1),box-shadow .2s ease,filter .2s ease; animation:winner-confirm-in .45s .35s both; }
.winner-confirm-label { position:relative; z-index:1; }
.winner-confirm-shine { position:absolute; inset:0; background:linear-gradient(105deg,transparent 25%,#ffffff88 50%,transparent 75%); transform:translateX(-130%); pointer-events:none; }
.winner-confirm:hover { transform:translateY(-4px) scale(1.045); box-shadow:0 14px 30px #2d789977,0 0 0 5px #7cc9e82a; filter:brightness(1.07); }
.winner-confirm:hover .winner-confirm-shine { animation:winner-confirm-sweep .7s ease-out; }
.winner-confirm:active { transform:translateY(1px) scale(.95); box-shadow:0 3px 10px #2d789944; }
.winner-confirm:focus-visible { outline:3px solid #6bc5e7; outline-offset:4px; }
@keyframes winner-confirm-in { from { opacity:0; transform:translateY(12px) scale(.96); } to { opacity:1; transform:translateY(0) scale(1); } }
@keyframes winner-confirm-sweep { from { transform:translateX(-130%); } to { transform:translateX(130%); } }
@media (prefers-reduced-motion:reduce) { .spin-button,.spin-button::before,.wheel-stage::after,.celebration-piece,.winner-confirm,.winner-confirm-shine { animation:none!important; } }

</style>
