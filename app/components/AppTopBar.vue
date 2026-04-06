<script setup lang="ts">
import type { ComponentPublicInstance } from 'vue'

const route = useRoute()
const colorMode = useColorMode()

const items = [
  { to: '/', label: 'Home' },
  { to: '/work', label: 'Work' },
  { to: '/contact', label: 'Contact' },
] as const

const navRowRef = ref<HTMLElement | null>(null)
const itemWrapRefs = ref<(HTMLElement | null)[]>([])

const underline = reactive({
  left: 0,
  width: 0,
  visible: false,
})

function setItemRef(el: Element | ComponentPublicInstance | null, index: number) {
  const node = el instanceof HTMLElement ? el : (el as ComponentPublicInstance | null)?.$el
  itemWrapRefs.value[index] = node instanceof HTMLElement ? node : null
}

const activeIndex = computed(() => {
  const path = route.path
  const idx = items.findIndex((item) =>
    item.to === '/' ? path === '/' : path === item.to || path.startsWith(`${item.to}/`),
  )
  return idx >= 0 ? idx : 0
})

function syncUnderline() {
  nextTick(() => {
    const nav = navRowRef.value
    const el = itemWrapRefs.value[activeIndex.value] ?? null
    if (!nav || !el) {
      underline.visible = false
      return
    }
    const nr = nav.getBoundingClientRect()
    const er = el.getBoundingClientRect()
    const extra = 12
    const halfExtra = extra / 2
    underline.width = er.width + extra
    underline.left = er.left - nr.left - halfExtra
    underline.visible = true
  })
}

watch(
  () => route.path,
  () => syncUnderline(),
  { immediate: true },
)

let resizeObserver: ResizeObserver | null = null

onMounted(() => {
  window.addEventListener('resize', syncUnderline)
  nextTick(() => {
    syncUnderline()
    if (typeof ResizeObserver !== 'undefined' && navRowRef.value) {
      resizeObserver = new ResizeObserver(() => syncUnderline())
      resizeObserver.observe(navRowRef.value)
    }
  })
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', syncUnderline)
  resizeObserver?.disconnect()
})

function toggleTheme() {
  colorMode.preference = colorMode.value === 'dark' ? 'light' : 'dark'
}
</script>

<template>
  <header class="top-bar">
    <div class="page-shell top-bar__inner">
      <nav class="nav" aria-label="Primary">
        <div ref="navRowRef" class="nav__row">
          <div
            v-for="(item, i) in items"
            :key="item.to"
            :ref="(el) => setItemRef(el, i)"
            class="nav__item-wrap"
          >
            <NuxtLink :to="item.to" class="nav__link text-ui">
              {{ item.label }}
            </NuxtLink>
          </div>
          <div
            class="nav__underline"
            :class="{ 'nav__underline--visible': underline.visible }"
            :style="{
              transform: `translateX(${underline.left}px)`,
              width: `${underline.width}px`,
            }"
          />
        </div>
      </nav>

      <button
        type="button"
        class="theme-toggle"
        :aria-label="colorMode.value === 'dark' ? 'Switch to light mode' : 'Switch to dark mode'"
        @click="toggleTheme"
      >
        <svg
          v-if="colorMode.value === 'light'"
          class="theme-toggle__icon"
          width="22"
          height="22"
          viewBox="0 0 24 24"
          fill="none"
          xmlns="http://www.w3.org/2000/svg"
          aria-hidden="true"
        >
          <path
            d="M21 14.5A7.5 7.5 0 0 1 9.5 3a7.5 7.5 0 1 0 11.5 11.5Z"
            stroke="currentColor"
            stroke-width="1.75"
            stroke-linecap="round"
            stroke-linejoin="round"
          />
        </svg>
        <svg
          v-else
          class="theme-toggle__icon"
          width="22"
          height="22"
          viewBox="0 0 24 24"
          fill="none"
          xmlns="http://www.w3.org/2000/svg"
          aria-hidden="true"
        >
          <circle cx="12" cy="12" r="4" stroke="currentColor" stroke-width="1.75" />
          <path
            d="M12 2v2M12 20v2M4.93 4.93l1.41 1.41M17.66 17.66l1.41 1.41M2 12h2M20 12h2M4.93 19.07l1.41-1.41M17.66 6.34l1.41-1.41"
            stroke="currentColor"
            stroke-width="1.75"
            stroke-linecap="round"
          />
        </svg>
      </button>
    </div>
  </header>
</template>

<style scoped>
.top-bar {
  position: sticky;
  top: 0;
  z-index: 50;
}

.top-bar__inner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 24px;
  min-height: 56px;
}

.theme-toggle {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 8px;
  margin: -8px;
  border: none;
  background: transparent;
  color: var(--color-main);
  cursor: pointer;
  border-radius: 6px;
}

.theme-toggle:focus-visible {
  outline: 2px solid var(--color-main);
  outline-offset: 2px;
}

.theme-toggle__icon {
  display: block;
}

.nav__row {
  position: relative;
  display: flex;
  align-items: center;
  gap: var(--nav-gap);
  padding-bottom: 4;
}

.nav__item-wrap {
  flex: 0 0 auto;
}

.nav__link {
  display: inline-block;
  text-decoration: none;
  color: var(--color-main);
  font-weight: var(--weight-medium);
  padding: 4px 0;
  opacity: 0.85;
  transition: opacity 0.15s ease;
}

.nav__link:hover,
.nav__link.router-link-active {
  opacity: 1;
}

.nav__underline {
  position: absolute;
  bottom: 0;
  left: 0;
  height: var(--nav-underline);
  background: var(--color-main);
  pointer-events: none;
  opacity: 0;
  transition:
    transform 0.25s cubic-bezier(0.22, 1, 0.36, 1),
    width 0.25s cubic-bezier(0.22, 1, 0.36, 1),
    opacity 0.15s ease;
}

.nav__underline--visible {
  opacity: 1;
}
</style>
