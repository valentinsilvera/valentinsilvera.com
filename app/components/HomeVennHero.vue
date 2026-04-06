<script setup lang="ts">
useHead({
  title: 'Home',
})

/** Depth + slight direction twist so parallax isn’t identical per circle. */
const PARALLAX_DEPTHS = [0.4, 0.62, 0.52] as const
/** Small rotation (rad) of the parallax vector per circle — keeps layers from moving in parallel. */
const PARALLAX_ROT_RAD = [0, 0.1, -0.085] as const

/** Smooth follow toward parallax target (avoids CSS transition restarts on every mousemove). */
const PARALLAX_LERP = 0.085

/** Fly-in: first 200ms keeps the old “snap in” speed; remaining ~800ms eases the tail smoothly. */
const INTRO_MS = 1500
const INTRO_FAST_MS = 300
/** Offset left after the fast phase (then bled to 0 over the long tail). */
const INTRO_TAIL_BLEND = 0.1
const INTRO_UNITS = [
  { x: 0, y: -1 },
  { x: -0.78, y: 0.72 },
  { x: 0.78, y: 0.72 },
] as const

function easeOutCubic(t: number) {
  return 1 - (1 - t) ** 1
}

function easeOutQuint(t: number) {
  return 1 - (1 - t) ** 5
}

/** 0 = settled, 1 = full intro offset. */
function introBlendAtElapsed(elapsed: number) {
  if (elapsed >= INTRO_MS) return 0
  if (elapsed <= INTRO_FAST_MS) {
    const u = elapsed / INTRO_FAST_MS
    const e = easeOutCubic(u)
    return 1 - e * (1 - INTRO_TAIL_BLEND)
  }
  const u = (elapsed - INTRO_FAST_MS) / (INTRO_MS - INTRO_FAST_MS)
  const e = easeOutQuint(u)
  return INTRO_TAIL_BLEND * (1 - e)
}

const sectionRef = ref<HTMLElement | null>(null)
const reduceMotion = ref(false)
/** 1 = full entrance offset, 0 = settled (multiplier on INTRO_UNITS × distance). */
const introBlend = ref(1)
const introDistance = ref(180)

let introRaf = 0
const parallaxTarget = reactive({ x: 0, y: 0 })
const parallaxCurrent = reactive({ x: 0, y: 0 })

const maxParallaxPx = ref(18)

let parallaxRaf = 0

function updateMaxParallax() {
  const el = sectionRef.value
  if (!el) return
  const w = el.getBoundingClientRect().width
  maxParallaxPx.value = Math.min(22, Math.max(10, w * 0.018))
}

function scheduleParallaxStep() {
  if (parallaxRaf !== 0 || reduceMotion.value) return
  parallaxRaf = requestAnimationFrame(parallaxStep)
}

function parallaxStep() {
  parallaxRaf = 0
  if (reduceMotion.value) return

  const tx = parallaxTarget.x
  const ty = parallaxTarget.y
  parallaxCurrent.x += (tx - parallaxCurrent.x) * PARALLAX_LERP
  parallaxCurrent.y += (ty - parallaxCurrent.y) * PARALLAX_LERP

  const err = Math.hypot(tx - parallaxCurrent.x, ty - parallaxCurrent.y)
  if (err < 0.02) {
    parallaxCurrent.x = tx
    parallaxCurrent.y = ty
  }

  const errAfter = Math.hypot(parallaxTarget.x - parallaxCurrent.x, parallaxTarget.y - parallaxCurrent.y)
  if (errAfter > 0.001) {
    scheduleParallaxStep()
  }
}

function cancelParallaxRaf() {
  if (parallaxRaf !== 0) {
    cancelAnimationFrame(parallaxRaf)
    parallaxRaf = 0
  }
}

function cancelIntroRaf() {
  if (introRaf !== 0) {
    cancelAnimationFrame(introRaf)
    introRaf = 0
  }
}

function updateIntroDistance() {
  if (typeof window === 'undefined') return
  const w = window.innerWidth
  const h = window.innerHeight
  introDistance.value = Math.min(240, Math.max(120, Math.min(w, h) * 0.28))
}

function introOffsetFor(index: 0 | 1 | 2) {
  const u = INTRO_UNITS[index]
  const s = introDistance.value * introBlend.value
  return { x: u.x * s, y: u.y * s }
}

function runHomeIntro() {
  cancelIntroRaf()
  introBlend.value = 1
  const t0 = performance.now()

  function tick(now: number) {
    const elapsed = now - t0
    if (elapsed >= INTRO_MS) {
      introBlend.value = 0
      introRaf = 0
      return
    }
    introBlend.value = introBlendAtElapsed(elapsed)
    introRaf = requestAnimationFrame(tick)
  }
  introRaf = requestAnimationFrame(tick)
}

function onSectionMove(e: MouseEvent) {
  const el = sectionRef.value
  if (!el) return
  const rect = el.getBoundingClientRect()
  const cx = rect.left + rect.width / 5
  const cy = rect.top + rect.height / 5
  const halfW = Math.max(rect.width / 2, 1)
  const halfH = Math.max(rect.height / 2, 1)
  let nx = (e.clientX - cx) / halfW
  let ny = (e.clientY - cy) / halfH
  nx = Math.max(-1, Math.min(1, nx))
  ny = Math.max(-1, Math.min(1, ny))
  const m = maxParallaxPx.value
  // Towards cursor (same magnitude as before, inverted)
  parallaxTarget.x = nx * m
  parallaxTarget.y = ny * m

  if (reduceMotion.value) {
    parallaxCurrent.x = parallaxTarget.x
    parallaxCurrent.y = parallaxTarget.y
    return
  }
  scheduleParallaxStep()
}

function onSectionLeave() {
  parallaxTarget.x = 0
  parallaxTarget.y = 0
  if (reduceMotion.value) {
    parallaxCurrent.x = 0
    parallaxCurrent.y = 0
    return
  }
  scheduleParallaxStep()
}

function circleParallaxDelta(index: 0 | 1 | 2) {
  const d = PARALLAX_DEPTHS[index]
  const t = PARALLAX_ROT_RAD[index]
  const bx = parallaxCurrent.x * d
  const by = parallaxCurrent.y * d
  const c = Math.cos(t)
  const s = Math.sin(t)
  return {
    dx: bx * c - by * s,
    dy: bx * s + by * c,
  }
}

function circleParallaxStyle(index: 0 | 1 | 2) {
  const { dx, dy } = circleParallaxDelta(index)
  const io = introOffsetFor(index)
  return {
    transform: `translate(calc(-50% + ${dx + io.x}px), calc(-50% + ${dy + io.y}px))`,
  }
}

/** Match centroid of the three circle centers so the name stays visually centered on the Venn. */
function titleParallaxStyle() {
  let sx = 0
  let sy = 0
  let ix = 0
  let iy = 0
  for (let i = 0; i < 3; i++) {
    const { dx, dy } = circleParallaxDelta(i as 0 | 1 | 2)
    sx += dx
    sy += dy
    const io = introOffsetFor(i as 0 | 1 | 2)
    ix += io.x
    iy += io.y
  }
  const ax = sx / 3 + ix / 3
  const ay = sy / 3 + iy / 3
  return {
    transform: `translate(calc(-50% + ${ax}px), calc(-50% + ${ay}px))`,
  }
}

let motionMq: MediaQueryList | null = null
let onMotionMqChange: (() => void) | null = null

onMounted(() => {
  motionMq = window.matchMedia('(prefers-reduced-motion: reduce)')
  reduceMotion.value = motionMq.matches
  onMotionMqChange = () => {
    if (!motionMq) return
    reduceMotion.value = motionMq.matches
    if (motionMq.matches) {
      cancelParallaxRaf()
      cancelIntroRaf()
      introBlend.value = 0
      parallaxTarget.x = 0
      parallaxTarget.y = 0
      parallaxCurrent.x = 0
      parallaxCurrent.y = 0
    }
  }
  motionMq.addEventListener('change', onMotionMqChange)
  updateMaxParallax()
  updateIntroDistance()
  window.addEventListener('resize', updateMaxParallax)
  window.addEventListener('resize', updateIntroDistance)

  if (reduceMotion.value) {
    introBlend.value = 0
  } else {
    runHomeIntro()
  }
})

onBeforeUnmount(() => {
  cancelParallaxRaf()
  cancelIntroRaf()
  if (motionMq && onMotionMqChange) {
    motionMq.removeEventListener('change', onMotionMqChange)
  }
  window.removeEventListener('resize', updateMaxParallax)
  window.removeEventListener('resize', updateIntroDistance)
})
</script>

<template>
  <section
    ref="sectionRef"
    class="home-venn"
    aria-labelledby="home-venn-title"
    @mousemove="onSectionMove"
    @mouseleave="onSectionLeave"
  >
    <div class="home-venn__inner">
      <div class="home-venn__stage" aria-hidden="true">
        <div class="venn-circle venn-circle--top" :style="circleParallaxStyle(0)">
          <div class="venn-circle__bob" :class="{ 'venn-circle__bob--intro-pending': introBlend > 0.001 }">
            <div class="venn-circle__disk" />
            <p class="venn-circle__label venn-circle__label--top font-mono text-ui">
              Product
            </p>
          </div>
        </div>

        <div class="venn-circle venn-circle--bl" :style="circleParallaxStyle(1)">
          <div class="venn-circle__bob" :class="{ 'venn-circle__bob--intro-pending': introBlend > 0.001 }">
            <div class="venn-circle__disk" />
            <p class="venn-circle__label venn-circle__label--bl font-mono text-ui" style=" padding: 0 24px;">
              <span class="home-venn__title-line">Software</span>
              <span class="home-venn__title-line">Engineering</span>
            </p>
          </div>
        </div>

        <div class="venn-circle venn-circle--br" :style="circleParallaxStyle(2)">
          <div class="venn-circle__bob" :class="{ 'venn-circle__bob--intro-pending': introBlend > 0.001 }">
            <div class="venn-circle__disk" />
            <p class="venn-circle__label venn-circle__label--br font-mono text-ui" style=" padding: 0 24px;">
              Design
            </p>
          </div>
        </div>
      </div>

      <h1 id="home-venn-title" class="home-venn__title font-mono" :style="titleParallaxStyle()">
        <span class="home-venn__title-line">valentin</span>
        <span class="home-venn__title-line">_silvera</span>
      </h1>
    </div>
  </section>
</template>

<style scoped>
.home-venn {
  --venn-d: min(500px, calc(100vw - 2 * var(--space-mobile-margin)));
  --venn-r: calc(var(--venn-d) / 2);
  /* Center distance as a fraction of diameter (triangle side length) */
  --venn-sep: calc(var(--venn-d) * 0.3);
  --venn-label-inset: 45px;
  /* Bubble drift amplitudes — scale with circle, stay very subtle (~2–4px at 500px) */
  --venn-float-1: calc(max(1px, var(--venn-d) * 0.02));
  --venn-float-2: calc(max(1.5px, var(--venn-d) * 0.03));
  --venn-float-3: calc(max(2px, var(--venn-d) * 0.04));
  /* √(3)/3 and √(3)/6 — equilateral triangle from circle centers */
  --venn-y-top: calc(var(--venn-sep) * 0.5773502691896258);
  --venn-y-base: calc(var(--venn-sep) * 0.28867513459481287);

  position: relative;
  width: 100%;
  min-height: clamp(420px, 65vh, 760px);
  display: flex;
  align-items: center;
  justify-content: center;
}

.home-venn__inner {
  position: relative;
  width: 100%;
  max-width: min(1120px, 100%);
  min-height: clamp(380px, 55vw, 640px);
}

.home-venn__stage {
  position: absolute;
  inset: 0;
  pointer-events: none;
}

.venn-circle {
  position: absolute;
  width: var(--venn-d);
  height: var(--venn-d);
  transform: translate(-50%, -50%);
  /* Parallax is smoothed in JS (rAF + lerp); no CSS transition on transform */
  will-change: transform;
}

.venn-circle--top {
  left: 50%;
  top: calc(50% - var(--venn-y-top));
}

.venn-circle--bl {
  left: calc(50% - var(--venn-sep) / 2);
  top: calc(50% + var(--venn-y-base));
}

.venn-circle--br {
  left: calc(50% + var(--venn-sep) / 2);
  top: calc(50% + var(--venn-y-base));
}

.venn-circle__bob {
  position: relative;
  width: 100%;
  height: 100%;
  border-radius: 50%;
  overflow: hidden;
  will-change: transform;
}

@media (prefers-reduced-motion: no-preference) {
  .venn-circle__bob--intro-pending {
    animation-play-state: paused;
  }

  /* Durations/delays/easing differ so loops rarely line up */
  .venn-circle--top .venn-circle__bob {
    animation: venn-bubble-a 34s cubic-bezier(0.42, 0.02, 0.58, 0.98) infinite;
    animation-delay: -2.5s;
  }

  .venn-circle--bl .venn-circle__bob {
    animation: venn-bubble-b 53s cubic-bezier(0.38, 0.08, 0.62, 0.92) infinite;
    animation-delay: -18.25s;
  }

  .venn-circle--br .venn-circle__bob {
    animation: venn-bubble-c 41s cubic-bezier(0.48, 0.04, 0.52, 0.96) infinite;
    animation-delay: -31.7s;
  }
}

/* Multi-stop paths: small translations only, loop smoothly (bubbles nudging in place) */
@keyframes venn-bubble-a {
  0%,
  100% {
    transform: translate3d(0, 0, 0);
  }
  16% {
    transform: translate3d(var(--venn-float-2), calc(-1 * var(--venn-float-3)), 0);
  }
  33% {
    transform: translate3d(calc(-1 * var(--venn-float-1)), calc(-1 * var(--venn-float-2)), 0);
  }
  50% {
    transform: translate3d(calc(-1 * var(--venn-float-3)), var(--venn-float-1), 0);
  }
  66% {
    transform: translate3d(calc(-1 * var(--venn-float-2)), var(--venn-float-3), 0);
  }
  83% {
    transform: translate3d(var(--venn-float-1), var(--venn-float-2), 0);
  }
}

@keyframes venn-bubble-b {
  0%,
  100% {
    transform: translate3d(0, 0, 0);
  }
  20% {
    transform: translate3d(calc(-1 * var(--venn-float-3)), var(--venn-float-2), 0);
  }
  40% {
    transform: translate3d(calc(-1 * var(--venn-float-1)), calc(-1 * var(--venn-float-2)), 0);
  }
  60% {
    transform: translate3d(var(--venn-float-2), calc(-1 * var(--venn-float-1)), 0);
  }
  80% {
    transform: translate3d(var(--venn-float-3), var(--venn-float-1), 0);
  }
}

@keyframes venn-bubble-c {
  0%,
  100% {
    transform: translate3d(0, 0, 0);
  }
  25% {
    transform: translate3d(var(--venn-float-3), var(--venn-float-1), 0);
  }
  50% {
    transform: translate3d(var(--venn-float-1), calc(-1 * var(--venn-float-3)), 0);
  }
  75% {
    transform: translate3d(calc(-1 * var(--venn-float-2)), calc(-1 * var(--venn-float-1)), 0);
  }
}

.venn-circle__disk {
  position: absolute;
  inset: 0;
  border-radius: 50%;
  background: color-mix(in srgb, var(--color-main) 10%, transparent);
}

html.dark .venn-circle__disk {
  background: color-mix(in srgb, var(--color-main) 14%, transparent);
}

.venn-circle__label {
  position: absolute;
  z-index: 1;
  margin: 0;
  font-weight: var(--weight-medium);
  color: var(--color-main);
  line-height: 1.25;
}

.venn-circle__label--top {
  left: 50%;
  top: var(--venn-label-inset);
  transform: translateX(-50%);
  text-align: center;
  max-width: calc(100% - 2 * var(--venn-label-inset));
}

/* Lower corners of the square bbox sit outside the disc — place copy in the lower lobes */
.venn-circle__label--bl {
  left: var(--venn-label-inset);
  bottom: calc(var(--venn-r) * 0.38);
  text-align: left;
  max-width: min(52%, calc(100% - var(--venn-label-inset) - var(--venn-r) * 0.15));
}

.venn-circle__label--br {
  right: var(--venn-label-inset);
  bottom: calc(var(--venn-r) * 0.38);
  text-align: right;
  max-width: min(52%, calc(100% - var(--venn-label-inset) - var(--venn-r) * 0.15));
}

.home-venn__title {
  position: absolute;
  left: 50%;
  top: 50%;
  /* transform from :style — averaged parallax to track circle centroid */
  z-index: 2;
  margin: 0;
  text-align: center;
  font-weight: 600;
  font-size: clamp(2rem, 5.5vw, var(--font-title-lg));
  line-height: 1.05;
  letter-spacing: -0.03em;
  color: var(--color-main);
  pointer-events: none;
  will-change: transform;
}

.home-venn__title-line {
  display: block;
}
</style>
