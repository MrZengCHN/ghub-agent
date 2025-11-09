<script setup>
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'

// Base logical size (only for initial viewBox mapping)
const VIEW_W = 800
const VIEW_H = 500
const GRID_EXTENT = 50000 // simulate "infinite" grid

const boardRef = ref(null)
const svgRef = ref(null)
const points = ref([]) // [{ id, x, y }]
const nextId = ref(1)

// Selection state (single + multi)
const selectedId = ref(null)
const selectedIds = ref(new Set())
const selectionCount = computed(() => selectedIds.value.size)

function isSelected(id) { return selectedIds.value.has(id) }
function setSingleSelection(id) {
  selectedId.value = id
  selectedIds.value = new Set(id != null ? [id] : [])
}
function toggleSelection(id) {
  const s = new Set(selectedIds.value)
  if (s.has(id)) s.delete(id); else s.add(id)
  selectedIds.value = s
  selectedId.value = s.size === 1 ? [...s][0] : null
}
function clearSelection() { setSingleSelection(null) }

const isDragging = ref(false)
const dragMode = ref(null) // 'point' | 'group' | null
const justDragged = ref(false) // prevent click-add right after a drag/pan

// Zoom & pan in WORLD units
const zoom = ref(1)
const MIN_ZOOM = 0.2
const MAX_ZOOM = 4
const ZOOM_STEP = 0.1
const panX = ref(0)
const panY = ref(0)

// Space-drag pan
const spaceHeld = ref(false)
const panning = ref(false)
const panStart = ref({ xr: 0, yr: 0, panX: 0, panY: 0 }) // starting root coords + pan

// Marquee selection (world coords)
const marquee = ref(null) // { x1, y1, x2, y2 }
const marqueeNorm = computed(() => {
  if (!marquee.value) return null
  const { x1, y1, x2, y2 } = marquee.value
  const x = Math.min(x1, x2), y = Math.min(y1, y2)
  const w = Math.abs(x2 - x1), h = Math.abs(y2 - y1)
  return { x, y, w, h }
})

// Relative input for adding points
const relDx = ref(0)
const relDy = ref(0)

// Deltas from previous point: [{dx, dy}]
const deltas = computed(() => {
  const arr = []
  for (let i = 1; i < points.value.length; i++) {
    const a = points.value[i - 1]
    const b = points.value[i]
    arr.push({ dx: Math.round(b.x - a.x), dy: Math.round(b.y - a.y) })
  }
  return arr
})

const selectedPoint = computed(() => points.value.find(p => p.id === selectedId.value) || null)

function clamp(v, lo, hi) { return Math.max(lo, Math.min(hi, v)) }

// Root <svg> CSS px to root user units (viewBox space) conversion
function rootScale() {
  const svg = svgRef.value
  const rect = svg.getBoundingClientRect()
  return { sx: rect.width / VIEW_W, sy: rect.height / VIEW_H, rect }
}
// Fixed-size point radius in screen pixels (shrunk)
const PT_RADIUS_PX = 3
// Track root SVG scale to keep point size constant across zoom and canvas resize
const rootSx = ref(1)
const rootSy = ref(1)
const ptR = computed(() => PT_RADIUS_PX / (zoom.value * (rootSx.value || 1)))
function updateRootScaleRef() {
  const svg = svgRef.value
  if (!svg) return
  const { sx, sy } = rootScale()
  rootSx.value = sx || 1
  rootSy.value = sy || 1
}

// Map client coords to WORLD coords (invert group transform: x_r = z*(x_w + panX))
function clientToWorld(evt) {
  const { sx, sy, rect } = rootScale()
  const xr = (evt.clientX - rect.left) / sx
  const yr = (evt.clientY - rect.top) / sy
  const x = xr / zoom.value - panX.value
  const y = yr / zoom.value - panY.value
  return { x, y, xr, yr }
}

function addPointAt(evt) {
  if (evt.altKey) return
  if (justDragged.value) { justDragged.value = false; return }
  if (spaceHeld.value || panning.value || marquee.value) return // disable add while panning/selection
  const { x, y } = clientToWorld(evt)
  const prev = points.value[points.value.length - 1]
  // Shift: new point keeps same x as previous, y follows cursor
  const nx = (evt.shiftKey && prev) ? prev.x : x
  const p = { id: nextId.value++, x: Math.round(nx), y: Math.round(y) }
  points.value.push(p)
  setSingleSelection(p.id)
}

// Drag bookkeeping for point/group moves
const dragStart = ref({ x: 0, y: 0, pts: [] })

function onPointDown(p, evt) {
  if (evt.altKey) return
  if (spaceHeld.value) return // space-pan mode: ignore point drag start
  evt.stopPropagation()
  if (evt.ctrlKey) {
    // toggle multi-select by ctrl-click
    toggleSelection(p.id)
    return
  }
  if (isSelected(p.id) && selectedIds.value.size > 1) {
    // group drag
    dragMode.value = 'group'
    const { x, y } = clientToWorld(evt)
    dragStart.value = { x, y, pts: [...selectedIds.value].map(id => {
      const q = points.value.find(pp => pp.id === id)
      return { id, x0: q.x, y0: q.y }
    }) }
  } else {
    // single drag
    setSingleSelection(p.id)
    dragMode.value = 'point'
  }
  isDragging.value = true
}

function onSvgMouseDown(evt) {
  if (evt.altKey) {
    evt.preventDefault()
    const { x, y } = clientToWorld(evt)
    bgPanning.value = true
    bgPanStart.value = { x, y, offX: bgOffX.value, offY: bgOffY.value }
    justDragged.value = true
    setTimeout(() => (justDragged.value = false), 0)
    return
  }
  // Space or middle-button triggers panning
  if (spaceHeld.value || evt.button === 1) {
    evt.preventDefault()
    const { xr, yr } = clientToWorld(evt)
    document.body.style.cursor = 'grabbing'
    panning.value = true
    panStart.value = { xr, yr, panX: panX.value, panY: panY.value }
    justDragged.value = true
    setTimeout(() => (justDragged.value = false), 0)
    return
  }
  // Ctrl+drag on empty area: marquee selection
  if (evt.ctrlKey) {
    const { x, y } = clientToWorld(evt)
    marquee.value = { x1: x, y1: y, x2: x, y2: y }
    justDragged.value = true
    setTimeout(() => (justDragged.value = false), 0)
    return
  }
}

function onSvgMouseMove(evt) {
  if (bgPanning.value) {
    const { x, y } = clientToWorld(evt)
    const dx = x - bgPanStart.value.x
    const dy = y - bgPanStart.value.y
    bgOffX.value = Math.round(bgPanStart.value.offX + dx)
    bgOffY.value = Math.round(bgPanStart.value.offY + dy)
    return
  }
  // Pan move
  if (panning.value) {
    const { xr, yr } = clientToWorld(evt)
    const dxr = xr - panStart.value.xr
    const dyr = yr - panStart.value.yr
    panX.value = panStart.value.panX + dxr / zoom.value
    panY.value = panStart.value.panY + dyr / zoom.value
    return
  }
  // Marquee update
  if (marquee.value) {
    const { x, y } = clientToWorld(evt)
    marquee.value.x2 = x
    marquee.value.y2 = y
    return
  }
  // Dragging points
  if (!isDragging.value) return
  if (dragMode.value === 'group') {
    const { x, y } = clientToWorld(evt)
    let dx = x - dragStart.value.x
    let dy = y - dragStart.value.y
    if (evt.shiftKey) { dx = 0 } // constrain to vertical
    for (const it of dragStart.value.pts) {
      const pt = points.value.find(pp => pp.id === it.id)
      pt.x = Math.round(it.x0 + dx)
      pt.y = Math.round(it.y0 + dy)
    }
    return
  }
  if (dragMode.value === 'point') {
    const { x, y } = clientToWorld(evt)
    const sp = selectedPoint.value
    if (!sp) return
    if (evt.shiftKey) {
      sp.y = Math.round(y)
    } else {
      sp.x = Math.round(x)
      sp.y = Math.round(y)
    }
  }
}

function onSvgMouseUp(evt) {
  if (bgPanning.value) { bgPanning.value = false }
  // Finish pan
  if (panning.value) {
    panning.value = false
    document.body.style.cursor = ''
  }
  // Finish marquee selection
  if (marquee.value) {
    const m = marqueeNorm.value
    marquee.value = null
    if (m && m.w > 0 && m.h > 0) {
      const inside = points.value.filter(p => p.x >= m.x && p.x <= m.x + m.w && p.y >= m.y && p.y <= m.y + m.h)
      selectedIds.value = new Set(inside.map(p => p.id))
      selectedId.value = selectedIds.value.size === 1 ? [...selectedIds.value][0] : null
    }
  }
  // Finish drag
  if (isDragging.value) {
    isDragging.value = false
    dragMode.value = null
    justDragged.value = true
    setTimeout(() => (justDragged.value = false), 0)
  }
}

function deleteSelected() {
  if (selectedIds.value.size === 0) return
  const ids = new Set(selectedIds.value)
  points.value = points.value.filter(p => !ids.has(p.id))
  clearSelection()
}

function clearAll() {
  points.value = []
  clearSelection()
  nextId.value = 1
}

// Zoom helpers using mouse pointer as the zoom center (in WORLD math)
function setZoomAround(z, clientX, clientY) {
  const { sx, sy, rect } = rootScale()
  const xr = (clientX - rect.left) / sx
  const yr = (clientY - rect.top) / sy
  const old = zoom.value
  const nz = Math.max(MIN_ZOOM, Math.min(MAX_ZOOM, Number(z)))
  // world coordinate under pointer
  const wx = xr / old - panX.value
  const wy = yr / old - panY.value
  zoom.value = nz
  // choose pan so that xr,yr unchanged after zoom
  panX.value = xr / nz - wx
  panY.value = yr / nz - wy
}
function zoomInAt(cx, cy) { setZoomAround(zoom.value + ZOOM_STEP, cx, cy) }
function zoomOutAt(cx, cy) { setZoomAround(zoom.value - ZOOM_STEP, cx, cy) }
function zoomIn() {
  const { rect } = rootScale()
  zoomInAt(rect.left + rect.width/2, rect.top + rect.height/2)
}
function zoomOut() {
  const { rect } = rootScale()
  zoomOutAt(rect.left + rect.width/2, rect.top + rect.height/2)
}
function resetZoom() { zoom.value = 1; panX.value = 0; panY.value = 0 }

function onWheel(evt) {
  if (evt.ctrlKey) {
    evt.preventDefault()
    const dir = Math.sign(evt.deltaY)
    if (dir > 0) zoomOutAt(evt.clientX, evt.clientY); else zoomInAt(evt.clientX, evt.clientY)
  }
}

// Keyboard: micro-adjust selection or selected point; Space toggles pan mode
function onKeydown(evt) {
  const tag = (evt.target && evt.target.tagName || '').toLowerCase()
  const typing = tag === 'input' || tag === 'textarea'
  if (evt.code === 'Space' && !typing) {
    spaceHeld.value = true
    document.body.style.cursor = 'grab'
    evt.preventDefault()
    return
  }
  if (typing) return
  const step = evt.shiftKey ? 10 : 1
  if (selectedIds.value.size > 1) {
    // move group
    for (const id of selectedIds.value) {
      const p = points.value.find(pp => pp.id === id)
      if (!p) continue
      switch (evt.key) {
        case 'ArrowUp':    p.y = Math.round(p.y - step); break
        case 'ArrowDown':  p.y = Math.round(p.y + step); break
        case 'ArrowLeft':  p.x = Math.round(p.x - step); break
        case 'ArrowRight': p.x = Math.round(p.x + step); break
        default: return
      }
    }
    evt.preventDefault()
    return
  }
  const p = selectedPoint.value
  if (!p) return
  switch (evt.key) {
    case 'ArrowUp':    p.y = Math.round(p.y - step); break
    case 'ArrowDown':  p.y = Math.round(p.y + step); break
    case 'ArrowLeft':  p.x = Math.round(p.x - step); break
    case 'ArrowRight': p.x = Math.round(p.x + step); break
    default: return
  }
  evt.preventDefault()
}
function onKeyup(evt) {
  if (evt.code === 'Space') {
    spaceHeld.value = false
    if (!panning.value) document.body.style.cursor = ''
  }
}

onMounted(() => {
  window.addEventListener('keydown', onKeydown)
  window.addEventListener('keyup', onKeyup)
  updateRootScaleRef()
  window.addEventListener('resize', updateRootScaleRef)
  // restore panel visibility
  const v = localStorage.getItem('ballistic_showPanel')
  if (v != null) showPanel.value = v === '1'
})

onBeforeUnmount(() => {
  window.removeEventListener('keydown', onKeydown)
  window.removeEventListener('keyup', onKeyup)
  window.removeEventListener('resize', updateRootScaleRef)
})

// Import/Export JSON
const ioText = ref('')
function exportToText() {
  const data = { points: points.value, deltas: deltas.value }
  ioText.value = JSON.stringify(data, null, 2)
}
function downloadJSON() {
  const data = ioText.value || JSON.stringify({ points: points.value, deltas: deltas.value }, null, 2)
  const blob = new Blob([data], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = 'ballistic.json'
  document.body.appendChild(a)
  a.click()
  a.remove()
  URL.revokeObjectURL(url)
}
function importFromText() {
  try {
    const obj = JSON.parse(ioText.value)
    const arr = Array.isArray(obj) ? obj : (obj.points || [])
    const cleaned = arr.map((p, i) => ({ id: p.id ?? i + 1, x: Math.round(Number(p.x)||0), y: Math.round(Number(p.y)||0) }))
    points.value = cleaned
    setSingleSelection(cleaned[0]?.id ?? null)
    nextId.value = (cleaned.reduce((m, p) => Math.max(m, p.id), 0) || 0) + 1
  } catch (e) {
    alert('导入失败：JSON 格式无效')
  }
}

// Add new point by relative displacement
function addByRelative() {
  const last = points.value[points.value.length - 1] || { x: 0, y: 0 }
  const nx = Math.round(last.x + Number(relDx.value || 0))
  const ny = Math.round(last.y + Number(relDy.value || 0))
  const p = { id: nextId.value++, x: nx, y: ny }
  points.value.push(p)
  setSingleSelection(p.id)
}

// SVG group transform matrix: world -> root user units
const vpMatrix = computed(() => `matrix(${zoom.value},0,0,${zoom.value},${zoom.value * panX.value},${zoom.value * panY.value})`)

// RPM -> Lua generation
const rpm = ref(600) // rounds per minute
const intervalMS = computed(() => { const r = Number(rpm.value||0); return r>0 ? Math.round(60000/r) : 0 })
const luaScript = computed(() => {
  const n = intervalMS.value
  if (!n || deltas.value.length === 0) return '-- 无可用位移或无效射速'
  const lines = ['-- 由 GHUB Agent 生成', '-- 射速(RPM) = ' + rpm.value, '-- 间隔 Sleep(ms) = ' + n]
  for (const d of deltas.value) {
    lines.push(`Sleep(${n})`)
    lines.push(`MoveMouseRelative(${d.dx}, ${d.dy})`)
  }
  return lines.join('\n')
})
function copyLua() {
  const txt = luaScript.value
  if (navigator.clipboard && navigator.clipboard.writeText) {
    navigator.clipboard.writeText(txt)
  } else {
    const ta = document.createElement('textarea')
    ta.value = txt
    document.body.appendChild(ta)
    ta.select()
    try { document.execCommand('copy') } finally { ta.remove() }
  }
}

// Layout toggles
const showPanel = ref(true)
const layoutClass = computed(() => showPanel.value ? '' : 'panel-hidden')
watch(showPanel, v => localStorage.setItem('ballistic_showPanel', v ? '1' : '0'))

// Delta selection (for buttons) + context menu + batch ops
const selectedDeltaIdxs = ref(new Set())
const lastDeltaClicked = ref(null)
const flashId = ref(null)

function deltaTargetPoint(i) { return points.value[i + 1] }
function deltaIsSelected(i) { return selectedDeltaIdxs.value.has(i) }
function syncPointsSelectionFromDelta() {
  const ids = new Set()
  for (const i of selectedDeltaIdxs.value) {
    const p = deltaTargetPoint(i)
    if (p) ids.add(p.id)
  }
  selectedIds.value = ids
  selectedId.value = ids.size === 1 ? [...ids][0] : null
}
function onDeltaClick(i, evt) {
  if (evt.shiftKey && lastDeltaClicked.value != null) {
    const a = Math.min(lastDeltaClicked.value, i)
    const b = Math.max(lastDeltaClicked.value, i)
    const set = new Set(selectedDeltaIdxs.value)
    for (let k = a; k <= b; k++) set.add(k)
    selectedDeltaIdxs.value = set
  } else if (evt.ctrlKey) {
    const set = new Set(selectedDeltaIdxs.value)
    if (set.has(i)) set.delete(i); else set.add(i)
    selectedDeltaIdxs.value = set
    lastDeltaClicked.value = i
  } else {
    selectedDeltaIdxs.value = new Set([i])
    lastDeltaClicked.value = i
  }
  syncPointsSelectionFromDelta()
}
function centerOnPoint(p) {
  const targetXR = VIEW_W / 2
  const targetYR = VIEW_H / 2
  panX.value = targetXR / zoom.value - p.x
  panY.value = targetYR / zoom.value - p.y
}
function selectDelta(i, options = {}) {
  onDeltaClick(i, { ctrlKey: false, shiftKey: false })
  const p = deltaTargetPoint(i)
  if (options.center && p) centerOnPoint(p)
  if (p) {
    flashId.value = p.id
    setTimeout(() => { if (flashId.value === p.id) flashId.value = null }, 300)
  }
}

const deltaMenu = ref({ show: false, x: 0, y: 0 })
function onDeltaContext(i, evt) {
  evt.preventDefault()
  if (!selectedDeltaIdxs.value.has(i)) {
    selectedDeltaIdxs.value = new Set([i])
    syncPointsSelectionFromDelta()
  }
  deltaMenu.value = { show: true, x: evt.clientX, y: evt.clientY }
}
function hideDeltaMenu() { deltaMenu.value.show = false }

const showDeltaEdit = ref(false)
const editDx = ref(0)
const editDy = ref(0)

function deleteSelectedDeltas() {
  if (selectedDeltaIdxs.value.size === 0) return
  const ids = new Set()
  for (const i of selectedDeltaIdxs.value) {
    const p = deltaTargetPoint(i)
    if (p) ids.add(p.id)
  }
  points.value = points.value.filter(p => !ids.has(p.id))
  selectedDeltaIdxs.value = new Set()
  clearSelection()
  hideDeltaMenu()
}
function openEditDialog() {
  editDx.value = 0
  editDy.value = 0
  showDeltaEdit.value = true
  hideDeltaMenu()
}
function applyDeltaEdit() {
  if (selectedDeltaIdxs.value.size === 0) { showDeltaEdit.value = false; return }
  const dx = Number(editDx.value || 0)
  const dy = Number(editDy.value || 0)
  for (const i of selectedDeltaIdxs.value) {
    const p = deltaTargetPoint(i)
    if (p) { p.x = Math.round(p.x + dx); p.y = Math.round(p.y + dy) }
  }
  showDeltaEdit.value = false
}

// Background image
const bgImageData = ref(null)
const bgInputRef = ref(null)
function chooseBg(){ if(bgInputRef.value) bgInputRef.value.click() }
function onBgFileChange(evt){ const f=evt.target.files?.[0]; if(!f) return; const r=new FileReader(); r.onload=()=>{ bgImageData.value = r.result }; r.readAsDataURL(f) }
function clearBg(){ bgImageData.value = null; if(bgInputRef.value) bgInputRef.value.value = '' }
// Background controls
const bgOpacity = ref(90) // 0-100
const bgScale = ref(100)  // 0-100 (%)
const bgOffX = ref(0)     // world units
const bgOffY = ref(0)     // world units
const bgPanning = ref(false)
const bgPanStart = ref({ x: 0, y: 0, offX: 0, offY: 0 })
const bgTransform = computed(() => "translate(" + bgOffX.value + ", " + bgOffY.value + ") scale(" + (bgScale.value/100) + ")")

</script>

<template>
  <div class="page">
    <header class="page-header">
      <h1 class="title">弹道绘制</h1>
      <div class="actions">
        <div class="zoom">
          <button class="btn" @click="zoomOut">-</button>
          <span class="zoom-val">{{ zoom.toFixed(2) }}x</span>
          <button class="btn" @click="zoomIn">+</button>
          <button class="btn ghost" @click="resetZoom">重置</button>
        </div>
        <div class="bg-actions">
          <input ref="bgInputRef" type="file" accept="image/*" class="hidden-file" @change="onBgFileChange" />
          <button class="btn" @click="chooseBg">上传背景</button>
          <button class="btn ghost" :disabled="!bgImageData" @click="clearBg">清除背景</button>
        </div>
        <button class="btn ghost" @click="showPanel = !showPanel">{{ showPanel ? '隐藏面板' : '显示面板' }}</button>
        <button class="btn" :disabled="selectionCount === 0" @click="deleteSelected">删除所选（{{ selectionCount }}）</button>
        <button class="btn" :disabled="points.length === 0" @click="clearAll">清空</button>
      </div>
    </header>

    <div class="layout" :class="layoutClass">
      <div class="board" ref="boardRef" @wheel="onWheel" @mousedown="onSvgMouseDown">
        <svg
          ref="svgRef"
          class="board-svg"
          :viewBox="`0 0 ${VIEW_W} ${VIEW_H}`"
          preserveAspectRatio="none"
          @click="addPointAt"
          @mousemove="onSvgMouseMove"
          @mouseup="onSvgMouseUp"
          @mouseleave="onSvgMouseUp"
        >
          <defs>
            <!-- minor grid: 1-unit; major: 10-unit -->
            <pattern id="grid10" width="10" height="10" patternUnits="userSpaceOnUse">
              <rect width="10" height="10" fill="none" />
              <!-- minor lines at every 1 unit -->
              <path d="M1 0 L1 10 M2 0 L2 10 M3 0 L3 10 M4 0 L4 10 M5 0 L5 10 M6 0 L6 10 M7 0 L7 10 M8 0 L8 10 M9 0 L9 10 M0 1 L10 1 M0 2 L10 2 M0 3 L10 3 M0 4 L10 4 M0 5 L10 5 M0 6 L10 6 M0 7 L10 7 M0 8 L10 8 M0 9 L10 9" fill="none" stroke="rgba(148,163,184,.15)" stroke-width="0.6" />
              <!-- major border each 10 units -->
              <path d="M 10 0 L 0 0 0 10 10 10 10 0" fill="none" stroke="rgba(148,163,184,.35)" stroke-width="1" />
            </pattern>
          </defs>

          <!-- World viewport group: pan/zoom via matrix; border outside remains fixed -->
          <g :transform="vpMatrix">
            <!-- background image -->
            <image v-if="bgImageData" :href="bgImageData" x="0" y="0" :width="VIEW_W" :height="VIEW_H" preserveAspectRatio="xMidYMid meet"  :opacity="bgOpacity/100" :transform="bgTransform" style="pointer-events:none;" />

            <!-- "Infinite" grid area -->
            <rect :x="-GRID_EXTENT" :y="-GRID_EXTENT" :width="GRID_EXTENT*2" :height="GRID_EXTENT*2" fill="url(#grid10)" />

            <!-- polyline between points -->
            <polyline v-if="points.length > 1" :points="points.map(p=>`${p.x},${p.y}`).join(' ')" class="line" />

            <!-- points -->
            <g v-for="p in points" :key="p.id">
              <circle class="pt" :class="{ sel: isSelected(p.id), flash_pt: flashId === p.id }" :cx="p.x" :cy="p.y" :r="ptR" @mousedown.prevent="onPointDown(p, $event)" />
              <text class="idx" :x="p.x + 10" :y="p.y - 10">{{ points.findIndex(x=>x.id===p.id) + 1 }}</text>
            </g>

            <!-- marquee rectangle (world coords) -->
            <rect v-if="marqueeNorm" class="marquee" :x="marqueeNorm.x" :y="marqueeNorm.y" :width="marqueeNorm.w" :height="marqueeNorm.h" />
          </g>
        </svg>
      </div>

      <div class="hotkeys">
        <div class="hotkeys-title">快捷键</div>
        <ul>
          <li>Ctrl + 滚轮：以指针为中心缩放</li>
          <li>空格 + 拖拽：平移画布（抓手）</li>
          <li>Shift + 拖拽：仅改 y；Shift + 点击：新点 x 与上一点相同</li>
          <li>Ctrl + 拖拽：框选点；Ctrl + 点击：切换选中</li>
          <li>方向键：移动 1；Shift + 方向键：移动 10</li>
        </ul>
      </div>

      <aside class="panel" v-show="showPanel">
        <div class="section">
          <div class="section-title">点信息</div>
          <div class="row">数量：<b>{{ points.length }}</b></div>
          <div class="row">已选：<b>{{ selectionCount }}</b></div>
          <div class="row" v-if="selectedPoint">单选：id {{ selectedPoint.id }} ({{ selectedPoint.x }}, {{ selectedPoint.y }})</div>
          <div class="muted">提示：Ctrl+拖拽框选；Ctrl+点切换选择；空格拖动画布；Shift 约束垂直移动</div>
        </div>
        <div class="section">
          <div class="section-title">位移数组 (dx, dy)</div>
          <div v-if="deltas.length === 0" class="muted">从第二个点开始记录位移</div>
          <div v-else class="delta-grid">
            <button
              v-for="(d,i) in deltas"
              :key="i"
              class="delta-btn"
              :class="{ active: deltaIsSelected(i), flash: points[i+1] && flashId === points[i+1].id }"
              @click="onDeltaClick(i, $event)"
              @dblclick.prevent="selectDelta(i, { center: true })"
              @contextmenu="onDeltaContext(i, $event)"
              title="单击选择/Shift区间/Ctrl多选；双击定位"
            >
              #{{ i+2 }} ← #{{ i+1 }} [{{ d.dx }}, {{ d.dy }}]
            </button>
          </div>
          <!-- Delta context menu -->
          <div v-if="deltaMenu.show" class="ctx-overlay" @click="hideDeltaMenu"></div>
          <div v-if="deltaMenu.show" class="ctx-menu" :style="{ left: deltaMenu.x + 'px', top: deltaMenu.y + 'px' }">
            <button class="ctx-item" @click="deleteSelectedDeltas">删除所选段（{{ selectedDeltaIdxs.size }}）</button>
            <button class="ctx-item" @click="openEditDialog">修改所选段…</button>
          </div>
        </div>
        <div class="section">
          <div class="section-title">输入相对位移添加新点</div>
          <div class="rel-inputs">
            <label>dx <input type="number" v-model.number="relDx" /></label>
            <label>dy <input type="number" v-model.number="relDy" /></label>
            <button class="btn" @click="addByRelative">添加</button>
          </div>
        </div>
        <div class="section">
          <div class="section-title">视图</div>
          <div class="range-wrap">
            <input type="range" min="0.2" max="4" step="0.01" v-model.number="zoom" />
            <span class="muted">缩放 {{ zoom.toFixed(2) }}x</span>
          </div>
        </div>
        <div class="section">
                  <div class="section">
          <div class="section-title">背景</div>
          <div class="range-wrap">
            <label>透明度 <input type="range" min="0" max="100" step="1" v-model.number="bgOpacity" /> <span class="muted">{{ bgOpacity }}%</span></label>
          </div>
          <div class="range-wrap">
            <label>大小 <input type="range" min="0" max="100" step="1" v-model.number="bgScale" /> <span class="muted">{{ bgScale }}%</span></label>
          </div>
          <div class="muted">提示：按住 Alt 拖动背景</div>
        </div><div class="section-title">压枪 Lua</div>
          <div class="rel-inputs">
            <label>射速 RPM <input type="number" min="1" v-model.number="rpm" /></label>
            <span class="muted">间隔 {{ intervalMS }} ms</span>
            <button class="btn" :disabled="deltas.length === 0 || intervalMS === 0" @click="copyLua">复制压枪Lua</button>
          </div>
          <pre class="code" style="max-height:160px; white-space:pre-wrap;">{{ luaScript }}</pre>
        </div>
        <div class="section">
          <div class="section-title">导入 / 导出 JSON</div>
          <div class="io">
            <textarea v-model="ioText" placeholder="在此粘贴或点击导出生成 JSON"></textarea>
            <div class="io-actions">
              <button class="btn" @click="exportToText">导出到文本</button>
              <button class="btn" @click="downloadJSON">下载 JSON</button>
              <button class="btn" :disabled="!ioText" @click="importFromText">导入</button>
            </div>
          </div>
        </div>
      </aside>

      <!-- Edit dialog -->
      <div v-if="showDeltaEdit" class="modal">
        <div class="modal-card">
          <div class="modal-title">批量修改所选段目标点坐标</div>
          <div class="rel-inputs">
            <label>Δx <input type="number" v-model.number="editDx" /></label>
            <label>Δy <input type="number" v-model.number="editDy" /></label>
          </div>
          <div class="modal-actions">
            <button class="btn" @click="applyDeltaEdit">应用</button>
            <button class="btn ghost" @click="showDeltaEdit=false">取消</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.page { display:flex; flex-direction:column; gap:16px; width:min(2200px, 100%); margin:0 auto; }
.page-header { display:flex; align-items:center; justify-content:space-between; flex-wrap:wrap; gap:12px; }
.title { margin:0; font-size:20px; font-weight:700; }
.actions { display:flex; gap:10px; align-items:center; flex-wrap:wrap; }
.bg-actions { display:flex; gap:8px; align-items:center; }
.hidden-file { display:none; }
.zoom { display:flex; align-items:center; gap:6px; background:#0b1020; border:1px solid #1f2937; padding:4px 6px; border-radius:8px; }
.zoom-val { color:#93c5fd; min-width:54px; text-align:center; }
.btn { background:#111827; color:#fff; border:1px solid #334155; padding:6px 10px; border-radius:8px; cursor:pointer; }
.btn.ghost { background: rgba(15,23,42,0.5); border-color:#334155; }
.btn.ghost:hover { background: rgba(15,23,42,0.7); }
.btn.ghost:disabled { opacity:.5; cursor:not-allowed; }
.btn:disabled { opacity:.5; cursor:not-allowed; }

.layout { --panel-w: clamp(300px, 22vw, 420px); display:grid; grid-template-columns: minmax(520px, 1fr) var(--panel-w); gap:16px; align-items:start; }
.layout.panel-hidden { grid-template-columns: 1fr; }
.board { width:100%; aspect-ratio: 16 / 10; border-radius:12px; border:1px solid #1f2937; background-color:#0b1220; overflow:hidden; position:relative; }
.board-svg { width:100%; height:100%; display:block; }
.line { fill:none; stroke:#22d3ee; stroke-width:2; }
.pt { fill:#3b82f6; stroke:#1e40af; stroke-width:2; cursor:grab; }
.pt.sel { fill:#22d3ee; stroke:#0891b2; }
.pt.flash_pt { animation: flashPt .3s linear 1; }
.idx { font-size:12px; fill:#e2e8f0; pointer-events:none; }

@keyframes flashPt { 0%{ filter:none } 50%{ filter: drop-shadow(0 0 6px #22d3ee) } 100%{ filter:none } }

.marquee { fill: rgba(59,130,246,0.15); stroke: #3b82f6; stroke-width: 1; shape-rendering: crispEdges; }

.panel { grid-column: 2; grid-row: 1; background:#0b1020; border:1px solid #1f2937; border-radius:12px; padding:12px; color:#e5e7eb; position:sticky; top:12px; max-height: calc(100vh - 180px); overflow:auto; }
.section { margin-bottom:12px; }
.section-title { font-weight:600; margin-bottom:6px; color:#93c5fd; }
.row { margin-bottom:4px; }
.muted { color:#94a3b8; font-size:12px; }
.delta-grid { display:flex; flex-wrap:wrap; gap:8px; max-height:180px; overflow:auto; }
.delta-btn { background:#0a0f1a; border:1px solid #1f2937; color:#e5e7eb; border-radius:999px; padding:4px 10px; cursor:pointer; font-size:12px; }
.delta-btn:hover { border-color:#334155; background:#0b1220; }
.delta-btn.active { border-color:#22d3ee; color:#93c5fd; }
.delta-btn.flash { animation: flashSel .3s linear 1; }
@keyframes flashSel { 0%{ box-shadow:0 0 0 0 rgba(34,211,238,0.0)} 50%{ box-shadow:0 0 0 4px rgba(34,211,238,0.25)} 100%{ box-shadow:0 0 0 0 rgba(34,211,238,0.0)} }

.rel-inputs { display:flex; gap:8px; align-items:center; flex-wrap:wrap; }
.rel-inputs input { width:110px; background:#0a0f1a; color:#e5e7eb; border:1px solid #1f2937; border-radius:8px; padding:6px 8px; }
.io textarea { width:100%; min-height:120px; resize:vertical; background:#0a0f1a; color:#e5e7eb; border:1px solid #1f2937; border-radius:8px; padding:8px; font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace; }
.io-actions { display:flex; gap:8px; margin-top:8px; }
.code { background:#0a0f1a; border:1px solid #1f2937; padding:8px; border-radius:8px; max-height:160px; overflow:auto; }

/* Range slider styling */
.range-wrap { display:flex; align-items:center; gap:10px; }
.panel input[type=range] { -webkit-appearance: none; width: 180px; height: 6px; background: linear-gradient(90deg,#0b1220,#0b1220); border:1px solid #1f2937; border-radius: 999px; outline: none; }
.panel input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; appearance: none; width: 14px; height: 14px; border-radius: 50%; background: linear-gradient(135deg,#22d3ee,#3b82f6); border:1px solid #0ea5e9; box-shadow: 0 0 0 2px rgba(14,165,233,0.2); cursor: pointer; }
.panel input[type=range]::-moz-range-thumb { width: 14px; height: 14px; border: none; border-radius: 50%; background: linear-gradient(135deg,#22d3ee,#3b82f6); box-shadow: 0 0 0 2px rgba(14,165,233,0.2); cursor: pointer; }
.panel input[type=range]::-webkit-slider-runnable-track { height: 6px; background: linear-gradient(90deg,#0b1220,#0b1220); border-radius: 999px; }
.panel input[type=range]::-moz-range-track { height: 6px; background: #0b1220; border-radius: 999px; }

.hotkeys { grid-column: 1 / span 1; margin-top:12px; background:#0b1020; border:1px solid #1f2937; border-radius:12px; padding:10px; color:#e5e7eb; }
.hotkeys-title { font-weight:600; margin-bottom:6px; color:#93c5fd; }
.hotkeys ul { margin:0; padding-left:18px; }
.hotkeys li { line-height:1.6; }

/* Context menu */
.ctx-overlay { position: fixed; inset: 0; background: transparent; z-index: 1000; }
.ctx-menu { position: fixed; z-index: 1001; background:#0b1020; border:1px solid #1f2937; border-radius:8px; padding:6px; min-width: 160px; box-shadow: 0 6px 18px rgba(0,0,0,0.4); }
.ctx-item { display:block; width:100%; text-align:left; background:transparent; border:none; color:#e5e7eb; padding:6px 10px; cursor:pointer; }
.ctx-item:hover { background:#111827; }

/* Modal */
.modal { position: fixed; inset: 0; background: rgba(2,6,23,0.6); display:flex; align-items:center; justify-content:center; z-index: 1100; }
.modal-card { width:min(420px, 90vw); background:#0b1020; border:1px solid #1f2937; border-radius:12px; padding:16px; color:#e5e7eb; }
.modal-title { font-weight:600; margin-bottom:10px; color:#93c5fd; }
.modal-actions { display:flex; gap:8px; justify-content:flex-end; margin-top:10px; }

@media (max-width: 1200px) {
  .layout { grid-template-columns: 1fr; }
  .panel { position:static; max-height:none; grid-column: auto; grid-row: auto; }
}

@media (min-width: 1800px) {
  .layout { --panel-w: clamp(340px, 20vw, 460px); }
}
</style>


/* Top bar: unify buttons to black */
.page-header .btn,
.page-header .btn.ghost { background:#000; border-color:#000; color:#fff; }
.page-header .btn:hover,
.page-header .btn.ghost:hover { background:#000; }
