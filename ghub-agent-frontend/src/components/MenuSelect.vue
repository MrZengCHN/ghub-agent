<script setup>
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'

const props = defineProps({
  modelValue: [String, Number, Boolean],
  options: { type: Array, default: () => [] }, // [{ value, label }]
  placeholder: { type: String, default: '请选择' },
  disabled: { type: Boolean, default: false },
})
const emit = defineEmits(['update:modelValue'])

const open = ref(false)
const activeIndex = ref(-1)
const rootEl = ref(null)

const selected = computed(() => props.options.find(o => o.value === props.modelValue))

function toggle(){ if (!props.disabled) open.value = !open.value }
function close(){ open.value = false; activeIndex.value = -1 }
function selectOption(opt){ emit('update:modelValue', opt.value); close() }

function onClickOutside(e){ if (!rootEl.value) return; if (!rootEl.value.contains(e.target)) close() }
function onKeydown(e){
  if (!open.value) {
    if (e.key === 'ArrowDown' || e.key === 'Enter' || e.key === ' ') { e.preventDefault(); open.value = true }
    return
  }
  if (e.key === 'Escape') { e.preventDefault(); close(); return }
  const max = props.options.length - 1
  if (e.key === 'ArrowDown') { e.preventDefault(); activeIndex.value = Math.min(max, (activeIndex.value < 0 ? 0 : activeIndex.value + 1)) }
  if (e.key === 'ArrowUp') { e.preventDefault(); activeIndex.value = Math.max(0, (activeIndex.value < 0 ? max : activeIndex.value - 1)) }
  if (e.key === 'Enter' || e.key === ' ') {
    e.preventDefault()
    const idx = activeIndex.value < 0 ? props.options.findIndex(o=>o.value===props.modelValue) : activeIndex.value
    if (idx >= 0) selectOption(props.options[idx])
  }
}

onMounted(() => { document.addEventListener('click', onClickOutside) })
onBeforeUnmount(() => { document.removeEventListener('click', onClickOutside) })
</script>

<template>
  <div ref="rootEl" class="relative inline-block w-full">
    <button type="button" class="w-full inline-flex items-center justify-between gap-2 px-3 py-2 rounded-lg border border-slate-200 bg-white hover:bg-slate-50 focus:outline-none focus:ring-2 focus:ring-indigo-500 disabled:opacity-60 disabled:cursor-not-allowed"
            :aria-expanded="open ? 'true' : 'false'" :disabled="disabled" @click="toggle" @keydown="onKeydown">
      <span class="truncate text-left flex-1">{{ selected?.label ?? placeholder }}</span>
      <svg class="shrink-0 text-slate-500" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="6 9 12 15 18 9"/></svg>
    </button>
    <div v-show="open" class="absolute z-30 mt-1 left-0 right-0 rounded-lg border border-slate-200 bg-white shadow-lg p-1">
      <ul role="menu" class="max-h-60 overflow-auto">
        <li v-for="(opt, i) in options" :key="String(opt.value)" role="menuitem"
            class="flex items-center gap-2 px-3 py-2 rounded-md cursor-pointer select-none"
            :class="[ (opt.value === modelValue) ? 'bg-slate-100 text-slate-900' : 'text-slate-800 hover:bg-slate-50' ]"
            @mouseenter="activeIndex = i" @mouseleave="activeIndex = -1" @click="selectOption(opt)">
          <svg v-if="opt.value === modelValue" class="text-indigo-600" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="20 6 9 17 4 12"/></svg>
          <span class="truncate">{{ opt.label }}</span>
        </li>
      </ul>
    </div>
  </div>
</template>

<style scoped>
/***** no-op; using Tailwind utilities *****/
</style>
