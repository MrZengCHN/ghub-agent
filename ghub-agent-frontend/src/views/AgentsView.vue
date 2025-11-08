<script setup>
import { reactive, ref, computed, watch, onMounted } from 'vue'
import MenuSelect from '@/components/MenuSelect.vue'
// Enums and labels
const MouseOnEvent = [
  { value: 'MOUSE_BUTTON_PRESSED', label: '鼠标按钮按下' },
  { value: 'MOUSE_BUTTON_RELEASED', label: '鼠标按钮抬起' },
]
const KeyboardEvent = [
  { value: 'PRESS', label: '按下' },
  { value: 'RELEASE', label: '抬起' },
  { value: 'PRESS_AND_RELEASE', label: '按下并抬起' },
]
const MouseButtonAction = [...KeyboardEvent]
const MacroEvent = [
  { value: 'PLAY', label: '播放' },
  { value: 'ABORT', label: '停止' },
  { value: 'PRESS', label: '按下' },
  { value: 'RELEASE', label: '释放' },
]
const MouseMoveEvent = [
  { value: 'MOVE_TO', label: '移动(绝对)' },
  { value: 'MOVE_TO_VIRTUAL', label: '移动(虚拟)' },
  { value: 'MOVE_RELATIVE', label: '相对移动' },
  { value: 'MOVE_WHEEL', label: '滚轮滚动' },
]

// Key codes (subset ordered, rest appended sorted)
const KeyCode = {
  ESCAPE:0x01,F1:0x3b,F2:0x3c,F3:0x3d,F4:0x3e,F5:0x3f,F6:0x40,F7:0x41,F8:0x42,F9:0x43,F10:0x44,F11:0x57,F12:0x58,
  F13:0x64,F14:0x65,F15:0x66,F16:0x67,F17:0x68,F18:0x69,F19:0x6a,F20:0x6b,F21:0x6c,F22:0x6d,F23:0x6e,F24:0x76,
  PRINTSCREEN:0x137,SCROLLLOCK:0x46,PAUSE:0x146,TILDE:0x29,DIGIT_1:0x02,DIGIT_2:0x03,DIGIT_3:0x04,DIGIT_4:0x05,DIGIT_5:0x06,DIGIT_6:0x07,DIGIT_7:0x08,DIGIT_8:0x09,DIGIT_9:0x0a,DIGIT_0:0x0b,MINUS:0x0c,EQUAL:0x0d,BACKSPACE:0x0e,
  TAB:0x0f,Q:0x10,W:0x11,E:0x12,R:0x13,T:0x14,Y:0x15,U:0x16,I:0x17,O:0x18,P:0x19,LBRACKET:0x1a,RBRACKET:0x1b,BACKSLASH:0x2b,
  CAPSLOCK:0x3a,A:0x1e,S:0x1f,D:0x20,F:0x21,G:0x22,H:0x23,J:0x24,K:0x25,L:0x26,SEMICOLON:0x27,QUOTE:0x28,ENTER:0x1c,
  LSHIFT:0x2a,NON_US_SLASH:0x56,Z:0x2c,X:0x2d,C:0x2e,V:0x2f,B:0x30,N:0x31,M:0x32,COMMA:0x33,PERIOD:0x34,SLASH:0x35,RSHIFT:0x36,
  LCTRL:0x1d,LGUI:0x15b,LALT:0x38,SPACEBAR:0x39,RALT:0x138,RGUI:0x15c,APPKEY:0x15d,RCTRL:0x11d,INSERT:0x152,HOME:0x147,
  PAGEUP:0x149,DELETE:0x153,END:0x14f,PAGEDOWN:0x151,UP:0x148,LEFT:0x14b,DOWN:0x150,RIGHT:0x14d,NUMLOCK:0x45,NUMSLASH:0x135,
  NUMMINUS:0x4a,NUM_7:0x47,NUM_8:0x48,NUM_9:0x49,NUMPLUS:0x4e,NUM_4:0x4b,NUM_5:0x4c,NUM_6:0x4d,NUM_1:0x4f,NUM_2:0x50,NUM_3:0x51,
  NUMENTER:0x11c,NUM_0:0x52,NUMPERIOD:0x53,
}
const preferred = ['LCTRL','RCTRL','LSHIFT','RSHIFT','LALT','RALT','CAPSLOCK','ESCAPE','ENTER','SPACEBAR','UP','DOWN','LEFT','RIGHT','A','S','D','W']
const KeyOptions = [
  ...preferred,
  ...Object.keys(KeyCode).filter(k => !preferred.includes(k)).sort(),
]
const KeyIndex = KeyOptions.map(k => ({
  key: k,
  code: KeyCode[k],
  hex: '0x' + (KeyCode[k] >>> 0).toString(16),
  label: `${k} (0x${(KeyCode[k] >>> 0).toString(16)})`,
  search: `${k.toLowerCase()} 0x${(KeyCode[k]>>>0).toString(16)} ${KeyCode[k]}`,
}))

const ModifierLower = { LSHIFT:'lshift', RSHIFT:'rshift', LCTRL:'lctrl', RCTRL:'rctrl', LALT:'lalt', RALT:'ralt' }
const LockLower = { CAPSLOCK:'capslock', NUMLOCK:'numlock', SCROLLLOCK:'scrolllock' }
const MODIFIER_KEYS = ['LCTRL','RCTRL','LSHIFT','RSHIFT','LALT','RALT']
const LOCK_KEYS = ['CAPSLOCK','NUMLOCK','SCROLLLOCK']

const LineTypeOptions = [
  { value: 'Keyboard', label: '键盘' },
  { value: 'Mouse', label: '鼠标' },
  { value: 'Sleep', label: '延时' },
  { value: 'Macro', label: '宏' },
  { value: 'MouseMove', label: '鼠标移动' },
]

// State
const builder = reactive({ mouseButtonBinds: [] })

function addBind() {
  builder.mouseButtonBinds.push({
    note: '',
    button: 1,
    mouseEvent: 'MOUSE_BUTTON_PRESSED',
    function: { functionName: 'MyFunc', lines: [] },
    collapsed: false,
    modifierModes: {}, // {KEY: 'ON'|'ANY'|'NOT'} - missing implies ANY
    lockModes: {},
  })
}
function removeBind(idx) { builder.mouseButtonBinds.splice(idx, 1) }
function toggleBind(idx) { const b = builder.mouseButtonBinds[idx]; b.collapsed = !b.collapsed }

// Tri-state helpers
function getMode(map, key) { return (map && map[key]) || 'ANY' }
function setMode(map, key, mode) { map[key] = mode }
function cycleMode(map, key) { const v = getMode(map, key); setMode(map, key, v === 'ANY' ? 'ON' : v === 'ON' ? 'NOT' : 'ANY') }
function modeClass(mode) {
  return mode === 'ON' ? 'border-emerald-200 bg-emerald-50 text-emerald-800'
    : mode === 'NOT' ? 'border-rose-200 bg-rose-50 text-rose-800'
    : 'border-slate-200 bg-slate-50 text-slate-700'
}

// JSON and Lua outputs
function jsonClean(b){
  return { mouseButtonBinds: (b.mouseButtonBinds||[]).map(x=>{
    const y = JSON.parse(JSON.stringify(x))
    delete y.collapsed
    // ensure maps
    y.modifierModes = y.modifierModes || {}
    y.lockModes = y.lockModes || {}
    // legacy arrays -> migrate
    if (Array.isArray(y.modifiers)) { const m={}; MODIFIER_KEYS.forEach(k=>{ if (y.modifiers.includes(k)) m[k]='ON' }); y.modifierModes = m; delete y.modifiers }
    if (Array.isArray(y.locks)) { const m={}; LOCK_KEYS.forEach(k=>{ if (y.locks.includes(k)) m[k]='ON' }); y.lockModes = m; delete y.locks }
    return y
  }) }
}
const jsonText = ref('')
watch(builder, () => { jsonText.value = JSON.stringify(jsonClean(builder), null, 2) }, { deep: true, immediate: true })

function asHex(n){ return '0x' + (n >>> 0).toString(16) }
function luaOfLine(ln) {
  if (ln.type === 'Keyboard') {
    const code = KeyCode[ln.key] ?? 0x00
    const fn = ln.keyboardEvent === 'PRESS' ? 'PressKey' : ln.keyboardEvent === 'RELEASE' ? 'ReleaseKey' : 'PressAndReleaseKey'
    return `${fn}(${asHex(code)})`
  }
  if (ln.type === 'Mouse') {
    const fn = ln.mouseEvent === 'PRESS' ? 'PressMouseButton' : ln.mouseEvent === 'RELEASE' ? 'ReleaseMouseButton' : 'PressAndReleaseMouseButton'
    return `${fn}(${Number(ln.mouseButtonNum) || 0})`
  }
  if (ln.type === 'Sleep') return `Sleep(${Number(ln.ms) || 0})`
  if (ln.type === 'Macro') {
    const name = ln.name ?? ''
    if (ln.macroEvent === 'PLAY') return `PlayMacro("${name}")`
    if (ln.macroEvent === 'ABORT') return `AbortMacro()`
    if (ln.macroEvent === 'PRESS') return `PressMacro("${name}")`
    if (ln.macroEvent === 'RELEASE') return `ReleaseMacro("${name}")`
    return ''
  }
  if (ln.type === 'MouseMove') {
    if (ln.moveEvent === 'MOVE_TO') return `MoveMouseTo(${Number(ln.x)||0}, ${Number(ln.y)||0})`
    if (ln.moveEvent === 'MOVE_TO_VIRTUAL') return `MoveMouseToVirtual(${Number(ln.x)||0}, ${Number(ln.y)||0})`
    if (ln.moveEvent === 'MOVE_RELATIVE') return `MoveMouseRelative(${Number(ln.x)||0}, ${Number(ln.y)||0})`
    if (ln.moveEvent === 'MOVE_WHEEL') return `MoveMouseWheel(${Number(ln.wheelSteps)||0})`
    return ''
  }
  return ''
}
function luaOfFunction(fn) {
  const lines = (fn.lines || []).map(l => `  ${luaOfLine(l)}`).join('\n')
  return `function ${fn.functionName || 'Unnamed'}()\n${lines}\nend\n`
}
function luaOfBind(b) {
  let s = ''
  const note = (b.note ?? '').trim()
  if (note) s += `     -- ${note}\n`
  s += '     if'
  let has=false
  if (b.mouseEvent) { s += ` event == "${b.mouseEvent}"`; has=true }
  if (Number(b.button) > 0) { s += (has ? ' and' : '') + ` arg == ${Number(b.button)}`; has=true }
  // modifiers tri-state
  const mm = b.modifierModes || {}
  MODIFIER_KEYS.forEach(k=>{
    const mode = mm[k] || 'ANY'; if (mode === 'ANY') return
    const lower = ModifierLower[k]
    s += ` and ${mode === 'ON' ? '' : 'not '}IsModifierPressed("${lower}")`
  })
  // locks tri-state
  const lm = b.lockModes || {}
  LOCK_KEYS.forEach(k=>{
    const mode = lm[k] || 'ANY'; if (mode === 'ANY') return
    const lower = LockLower[k]
    s += ` and ${mode === 'ON' ? '' : 'not '}IsKeyLockOn("${lower}")`
  })
  s += " then\n"
  if (b.function && (b.function.functionName || '').length > 0) s += `         ${b.function.functionName}()\n`
  s += '     end'
  return s
}
const luaText = computed(() => {
  const binds = builder.mouseButtonBinds || []
  let out = ''
  out += 'EnablePrimaryMouseButtonEvents(true)\n'
  out += 'function OnEvent(event, arg)'
  for (const b of binds) { out += '\n' + luaOfBind(b) + '\n' }
  out += 'end\n'
  for (const b of binds) { if (b.function) out += '\n' + luaOfFunction(b.function) + '\n' }
  return out
})

// Edits
function updateBind(idx, field, value) {
  const b = builder.mouseButtonBinds[idx]
  if (field === 'button') b.button = Number(value)
  else if (field === 'mouseEvent') b.mouseEvent = value
  else if (field === 'note') b.note = value
  else if (field === 'functionName') b.function.functionName = value
}
function updateLine(idx, lidx, field, value) {
  const ln = builder.mouseButtonBinds[idx].function.lines[lidx]
  if (['mouseButtonNum','ms','x','y','wheelSteps'].includes(field)) ln[field] = Number(value)
  else ln[field] = value
}

function addLine(idx, type) {
  const fn = builder.mouseButtonBinds[idx].function
  const ln = { type }
  if (type === 'Keyboard') { ln.keyboardEvent = 'PRESS'; ln.key = 'A' }
  else if (type === 'Mouse') { ln.mouseEvent = 'PRESS'; ln.mouseButtonNum = 1 }
  else if (type === 'Sleep') { ln.ms = 30 }
  else if (type === 'Macro') { ln.macroEvent = 'PLAY'; ln.name = 'MyMacro' }
  else if (type === 'MouseMove') { ln.moveEvent='MOVE_RELATIVE'; ln.x=10; ln.y=-5; ln.wheelSteps=0 }
  fn.lines.push(ln)
}
function removeLine(idx, lidx) { builder.mouseButtonBinds[idx].function.lines.splice(lidx, 1) }

// Import/Export helpers
function loadFromJson() {
  try {
    const obj = JSON.parse(jsonText.value || '{"mouseButtonBinds":[]}')
    const arr = Array.isArray(obj.mouseButtonBinds) ? obj.mouseButtonBinds : []
    builder.mouseButtonBinds = arr.map(x => {
      const y = { ...x }
      y.collapsed = false
      if (!y.modifierModes) { const m = {}; (y.modifiers||[]).forEach(k=> m[k]='ON'); y.modifierModes = m; delete y.modifiers }
      if (!y.lockModes) { const m = {}; (y.locks||[]).forEach(k=> m[k]='ON'); y.lockModes = m; delete y.locks }
      return y
    })
  } catch (e) {
    alert('JSON 解析失败: ' + e.message)
  }
}
function copyToClipboard(text) {
  if (navigator.clipboard?.writeText) return navigator.clipboard.writeText(text)
  const ta = document.createElement('textarea'); ta.value = text; document.body.appendChild(ta); ta.select(); document.execCommand('copy'); ta.remove();
  return Promise.resolve()
}
function copyJson() { copyToClipboard(jsonText.value).then(() => toast('JSON 已复制')) }
function downloadJson() {
  const blob = new Blob([jsonText.value], { type: 'application/json;charset=utf-8' })
  const a = document.createElement('a'); a.href = URL.createObjectURL(blob); a.download = 'ghub-config.json'; a.click(); URL.revokeObjectURL(a.href)
}
function copyLua() { copyToClipboard(luaText.value).then(() => toast('Lua 已复制')) }
function downloadLua() {
  const blob = new Blob([luaText.value], { type: 'text/plain;charset=utf-8' })
  const a = document.createElement('a'); a.href = URL.createObjectURL(blob); a.download = 'script.lua'; a.click(); URL.revokeObjectURL(a.href)
}

// Tiny toast
const toastMsg = ref('')
const toastShow = ref(false)
function toast(msg) { toastMsg.value = msg; toastShow.value = true; setTimeout(() => toastShow.value = false, 1600) }

onMounted(() => { if (!builder.mouseButtonBinds.length) addBind() })
</script>

<template>
  <div class="w-full mx-auto px-4 sm:px-6 lg:px-8 max-w-[1200px] xl:max-w-[1600px] 2xl:max-w-[2000px] space-y-4">
    <div class="flex items-center justify-between">
      <h1 class="text-xl font-semibold text-slate-900">按键绑定生成器</h1>
      <div class="flex gap-2">
        <button class="px-3 py-2 rounded-lg border border-slate-200 bg-white hover:bg-slate-50" @click="builder.mouseButtonBinds = []">清空</button>
        <button class="px-3 py-2 rounded-lg border border-indigo-200 text-indigo-800 bg-indigo-50 hover:bg-indigo-100" @click="addBind">新增绑定</button>
      </div>
    </div>

    <div class="grid grid-cols-1 md:grid-cols-2 items-start gap-4 xl:gap-6 2xl:gap-8 xl:[grid-template-columns:1.5fr_1fr] 2xl:[grid-template-columns:2fr_1fr]">
      <!-- Left: Binds -->
      <section class="bg-white border border-slate-200 rounded-xl p-3 shadow-sm">
        <h2 class="text-sm font-medium text-slate-800 mb-2">按键绑定</h2>
        <div class="space-y-3">
          <div v-for="(b, idx) in builder.mouseButtonBinds" :key="idx" class="rounded-lg border border-slate-200 bg-slate-50">
            <!-- Header -->
            <div class="flex items-center gap-2 p-2">
              <span class="inline-flex items-center gap-2 text-xs px-2 py-1 rounded-full border border-indigo-200 bg-indigo-50 text-indigo-800">绑定 #{{ idx + 1 }}</span>
              <div class="flex-1"></div>
              <button class="hidden sm:inline-flex w-[88px] h-[28px] items-center justify-center text-xs rounded border border-indigo-200 bg-indigo-50 text-indigo-800 hover:bg-indigo-100" @click="builder.mouseButtonBinds.splice(idx,1)">删除绑定</button>
              <button class="w-[88px] h-[28px] inline-flex items-center justify-center text-xs rounded border border-indigo-200 bg-indigo-50 text-indigo-800 hover:bg-indigo-100 sm:hidden" @click="builder.mouseButtonBinds.splice(idx,1)">删除绑定</button>
              <button class="w-[88px] h-[28px] inline-flex items-center justify-center text-xs rounded border border-indigo-200 bg-indigo-50 text-indigo-800 hover:bg-indigo-100" :aria-label="b.collapsed ? '展开' : '收起'" @click="toggleBind(idx)">
                <svg class="transition-transform" :class="b.collapsed ? 'rotate-0' : 'rotate-90'" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" width="12" height="12"><polyline points="9 18 15 12 9 6"/></svg>
              </button>
            </div>
            <!-- Summary when collapsed -->
            <div v-if="b.collapsed" class="px-3 pb-2 text-xs text-slate-500 truncate">
              {{ [b.note && b.note.trim(), b.button ? ('arg:' + b.button) : '', b.mouseEvent, b.function?.functionName, (b.function?.lines?.length||0) ? ('行:' + (b.function?.lines?.length||0)) : ''].filter(Boolean).join(' · ') }}
            </div>

            <!-- Body -->
            <div v-show="!b.collapsed" class="p-3 space-y-3 border-t border-slate-200 bg-white rounded-b-lg">
              <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-3">
                <div>
                  <label class="block text-[12px] text-slate-600 mb-1 flex items-center gap-1">备注</label>
                  <input type="text" class="w-full px-3 py-2 rounded-lg border border-slate-200" :value="b.note||''" @input="e=>updateBind(idx,'note',e.target.value)" />
                </div>
                <div>
                  <label class="block text-[12px] text-slate-600 mb-1 flex items-center gap-1">绑定按键 (arg)
                    <span class="relative group inline-flex items-center justify-center w-[18px] h-[18px] rounded-full border border-indigo-200 bg-indigo-50 text-indigo-800 text-[12px] leading-none cursor-help" tabindex="0" aria-label="说明">i
                      <span class="pointer-events-none absolute left-1/2 top-[calc(100%+8px)] z-20 -translate-x-1/2 break-words rounded-md border border-slate-200 bg-white px-2 py-1 text-[12px] text-slate-700 shadow-xl opacity-0 scale-95 transition-all duration-150 group-hover:opacity-100 group-hover:scale-100 group-focus-visible:opacity-100 group-focus-visible:scale-100">日志中的 2 代表鼠标右键；在“函数体行”中 2 代表鼠标中键
                        <span class="absolute -top-1 left-1/2 -translate-x-1/2 h-2 w-2 rotate-45 bg-white border-l border-t border-slate-200"></span>
                      </span>
                    </span>
                  </label>
                  <input type="number" min="1" class="w-full px-3 py-2 rounded-lg border border-slate-200" :value="Number(b.button)||1" @input="e=>updateBind(idx,'button',e.target.value)" />
                </div>
                <div>
                  <label class="block text-[12px] text-slate-600 mb-1 flex items-center gap-1">事件 (event)</label>
                  <MenuSelect :options="MouseOnEvent" :model-value="b.mouseEvent" @update:modelValue="v=>updateBind(idx,'mouseEvent',v)" />
                </div>
                <div>
                  <label class="block text-[12px] text-slate-600 mb-1 flex items-center gap-1">函数名</label>
                  <input type="text" class="w-full px-3 py-2 rounded-lg border border-slate-200" :value="b.function?.functionName||''" @input="e=>updateBind(idx,'functionName',e.target.value)" />
                </div>
              </div>

              <div class="grid grid-cols-1 sm:grid-cols-2 gap-3">
                <div>
                  <div class="flex items-center justify-between mb-1"><label class="block text-[12px] text-slate-600">修饰键</label><div class="flex items-center gap-2 text-[11px] text-slate-600"><span class="inline-flex items-center gap-1"><span class="w-2.5 h-2.5 rounded-full bg-emerald-500 border border-emerald-600"></span>ON</span><span class="inline-flex items-center gap-1"><span class="w-2.5 h-2.5 rounded-full bg-slate-400 border border-slate-500"></span>ANY</span><span class="inline-flex items-center gap-1"><span class="w-2.5 h-2.5 rounded-full bg-rose-500 border border-rose-600"></span>NOT</span></div></div>
                  <div class="flex flex-wrap gap-2">
                    <button v-for="m in MODIFIER_KEYS" :key="m" type="button" class="flex items-start gap-2 px-2 py-1 rounded-lg border text-[12px] grow basis-1/2 xl:basis-1/3 min-w-[120px] max-w-full overflow-hidden" :class="modeClass(getMode(b.modifierModes, m))" @click="cycleMode(b.modifierModes, m)">
  <span class="flex-1 min-w-0 break-words text-left">{{ m }}</span>
</button>
                  </div>
                </div>
                <div>
                  <div class="flex items-center justify-between mb-1"><label class="block text-[12px] text-slate-600">锁定键</label><div class="flex items-center gap-2 text-[11px] text-slate-600"><span class="inline-flex items-center gap-1"><span class="w-2.5 h-2.5 rounded-full bg-emerald-500 border border-emerald-600"></span>ON</span><span class="inline-flex items-center gap-1"><span class="w-2.5 h-2.5 rounded-full bg-slate-400 border border-slate-500"></span>ANY</span><span class="inline-flex items-center gap-1"><span class="w-2.5 h-2.5 rounded-full bg-rose-500 border border-rose-600"></span>NOT</span></div></div>
                  <div class="flex flex-wrap gap-2">
                    <button v-for="k in LOCK_KEYS" :key="k" type="button" class="flex items-start gap-2 px-2 py-1 rounded-lg border text-[12px] grow basis-1/2 xl:basis-1/3 min-w-[120px] max-w-full overflow-hidden" :class="modeClass(getMode(b.lockModes, k))" @click="cycleMode(b.lockModes, k)">
  <span class="flex-1 min-w-0 break-words text-left">{{ k }}</span>
</button>
                  </div>
                </div>
              </div>

              <div>
                <h3 class="text-sm font-medium text-slate-700 mb-2">函数体行</h3>
                <div class="space-y-2">
                  <div v-for="(ln,lidx) in b.function?.lines || []" :key="lidx" class="p-2 rounded-lg border border-slate-200 bg-white">
                    <!-- Line editors by type -->
                    <div v-if="ln.type==='Keyboard'" class="grid grid-cols-1 sm:grid-cols-3 gap-3 items-end">
                      <div>
                        <label class="block text-[12px] text-slate-600 mb-1">事件</label>

                        <MenuSelect :options="KeyboardEvent" :model-value="ln.keyboardEvent" @update:modelValue="v=>updateLine(idx,lidx,'keyboardEvent',v)" />

                      </div>
                      <div>
                        <label class="block text-[12px] text-slate-600 mb-1">按键</label>
                        <div>
                          <input class="w-full px-3 py-2 rounded-lg border border-slate-200" :value="ln.key || ''" @input="e=>updateLine(idx,lidx,'key',e.target.value)" list="key-picker" placeholder="搜索键名或代码，如 A/0x1d/ctrl" />
                        </div>
                      </div>
                      <div class="text-right">
                        <button class="px-3 py-2 rounded-lg border border-slate-200 bg-slate-50 hover:bg-slate-100" @click="removeLine(idx,lidx)">删除</button>
                      </div>
                    </div>

                    <div v-else-if="ln.type==='Mouse'" class="grid grid-cols-1 sm:grid-cols-3 gap-3 items-end">
                      <div>
                        <label class="block text-[12px] text-slate-600 mb-1">事件</label>
                        <MenuSelect :options="MouseButtonAction" :model-value="ln.mouseEvent" @update:modelValue="v=>updateLine(idx,lidx,'mouseEvent',v)" />
                      </div>
                      <div>
                        <label class="block text-[12px] text-slate-600 mb-1">按钮编号</label>
                        <input type="number" class="w-full px-3 py-2 rounded-lg border border-slate-200" :value="ln.mouseButtonNum ?? 1" @input="e=>updateLine(idx,lidx,'mouseButtonNum',e.target.value)" />
                      </div>
                      <div class="text-right">
                        <button class="px-3 py-2 rounded-lg border border-slate-200 bg-slate-50 hover:bg-slate-100" @click="removeLine(idx,lidx)">删除</button>
                      </div>
                    </div>

                    <div v-else-if="ln.type==='Sleep'" class="grid grid-cols-1 sm:grid-cols-3 gap-3 items-end">
                      <div>
                        <label class="block text-[12px] text-slate-600 mb-1">毫秒</label>
                        <input type="number" class="w-full px-3 py-2 rounded-lg border border-slate-200" :value="ln.ms ?? 30" @input="e=>updateLine(idx,lidx,'ms',e.target.value)" />
                      </div>
                      <div class="text-right sm:col-span-2">
                        <button class="px-3 py-2 rounded-lg border border-slate-200 bg-slate-50 hover:bg-slate-100" @click="removeLine(idx,lidx)">删除</button>
                      </div>
                    </div>

                    <div v-else-if="ln.type==='Macro'" class="grid grid-cols-1 sm:grid-cols-3 gap-3 items-end">
                      <div>
                        <label class="block text-[12px] text-slate-600 mb-1">事件</label>
                        <MenuSelect :options="MacroEvent" :model-value="ln.macroEvent" @update:modelValue="v=>updateLine(idx,lidx,'macroEvent',v)" />
                      </div>
                      <div>
                        <label class="block text-[12px] text-slate-600 mb-1">名称</label>
                        <input type="text" class="w-full px-3 py-2 rounded-lg border border-slate-200" :value="ln.name ?? ''" @input="e=>updateLine(idx,lidx,'name',e.target.value)" />
                      </div>
                      <div class="text-right">
                        <button class="px-3 py-2 rounded-lg border border-slate-200 bg-slate-50 hover:bg-slate-100" @click="removeLine(idx,lidx)">删除</button>
                      </div>
                    </div>

                    <div v-else-if="ln.type==='MouseMove'" class="grid grid-cols-1 sm:grid-cols-5 gap-3 items-end">
                      <div class="sm:col-span-2">
                        <label class="block text-[12px] text-slate-600 mb-1">事件</label>
                        <MenuSelect :options="MouseMoveEvent" :model-value="ln.moveEvent" @update:modelValue="v=>updateLine(idx,lidx,'moveEvent',v)" />
                      </div>
                      <div v-if="ln.moveEvent!=='MOVE_WHEEL'">
                        <label class="block text-[12px] text-slate-600 mb-1">X</label>
                        <input type="number" class="w-full px-3 py-2 rounded-lg border border-slate-200" :value="ln.x ?? 0" @input="e=>updateLine(idx,lidx,'x',e.target.value)" />
                      </div>
                      <div v-if="ln.moveEvent!=='MOVE_WHEEL'">
                        <label class="block text-[12px] text-slate-600 mb-1">Y</label>
                        <input type="number" class="w-full px-3 py-2 rounded-lg border border-slate-200" :value="ln.y ?? 0" @input="e=>updateLine(idx,lidx,'y',e.target.value)" />
                      </div>
                      <div v-else>
                        <label class="block text-[12px] text-slate-600 mb-1">滚轮步数</label>
                        <input type="number" class="w-full px-3 py-2 rounded-lg border border-slate-200" :value="ln.wheelSteps ?? 0" @input="e=>updateLine(idx,lidx,'wheelSteps',e.target.value)" />
                      </div>
                      <div class="text-right">
                        <button class="px-3 py-2 rounded-lg border border-slate-200 bg-slate-50 hover:bg-slate-100" @click="removeLine(idx,lidx)">删除</button>
                      </div>
                    </div>
                  </div>
                </div>

                <div class="flex items-center gap-2 mt-2">
<div class="basis-1/2 min-w-[160px]">
                  <MenuSelect :options="LineTypeOptions" :model-value="b._newType || 'Keyboard'" @update:modelValue="v => b._newType = v" />
</div>
                  <button class="ml-auto px-3 py-2 rounded-lg border border-slate-200 bg-white hover:bg-slate-50" @click="addLine(idx, b._newType || 'Keyboard')">添加行</button>
                </div>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- Right: JSON + Lua outputs -->
      <section class="xl:sticky xl:top-6 h-fit bg-white border border-slate-200 rounded-xl p-3 shadow-sm">
        <h2 class="text-sm font-medium text-slate-800 mb-2">JSON · 生成 · 导出</h2>
        <label class="block text-[12px] text-slate-600 mb-1">配置 JSON</label>
        <textarea class="w-full min-h-[200px] md:min-h-[240px] max-h-[60vh] xl:max-h-[65vh] 2xl:max-h-[70vh] resize-y px-3 py-2 rounded-lg border border-slate-200 font-mono text-[12px]" v-model="jsonText" spellcheck="false"></textarea>
        <div class="flex flex-wrap gap-2 my-2">
          <button class="px-3 py-2 rounded-lg border border-slate-200 bg-white hover:bg-slate-50" @click="loadFromJson">从 JSON 载入</button>
          <button class="px-3 py-2 rounded-lg border border-slate-200 bg-white hover:bg-slate-50" @click="copyJson">复制 JSON</button>
          <button class="px-3 py-2 rounded-lg border border-slate-200 bg-white hover:bg-slate-50" @click="downloadJson">下载 .json</button>
          <button class="px-3 py-2 rounded-lg border border-indigo-200 text-indigo-800 bg-indigo-50 hover:bg-indigo-100" @click="null">生成 Lua（离线）</button>
          <button class="px-3 py-2 rounded-lg border border-slate-200 bg-white hover:bg-slate-50" @click="copyLua">复制 Lua</button>
          <button class="px-3 py-2 rounded-lg border border-slate-200 bg-white hover:bg-slate-50" @click="downloadLua">下载 .lua</button>
        </div>
        <label class="block text-[12px] text-slate-600 mb-1">Lua 输出</label>
        <div class="rounded-lg border border-slate-200 bg-slate-900 text-slate-100 overflow-auto max-h-[420px] md:max-h-[50vh] xl:max-h-[60vh] 2xl:max-h-[65vh]">
          <pre class="p-3 text-[12px] whitespace-pre">{{ luaText }}</pre>
        </div>
        <div class="text-[12px] text-slate-500 mt-2">注：本页面完全在浏览器中构建 Lua，无需后端接口。</div>
      </section>
    </div>

    <!-- global datalist for key picker -->
    <datalist id="key-picker">
      <option v-for="o in KeyIndex" :key="o.key" :value="o.key" :label="o.label"></option>
    </datalist>

    <!-- toast -->
    <div v-show="toastShow" class="fixed bottom-4 left-1/2 -translate-x-1/2 bg-slate-900 text-white px-3 py-2 rounded-lg shadow" role="status">{{ toastMsg }}</div>
  </div>
</template>

<style scoped>
.rotate-0 { transform: rotate(0deg); }
.rotate-90 { transform: rotate(90deg); }
</style>










