<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { useRafFn } from '@vueuse/core'
import { Icon } from '@iconify/vue'
import { RouterLink } from 'vue-router'

// ── Circuit parameters ──────────────────────────────────────────────────────
const R = ref(100) // Ohms
const L = ref(100) // mH
const C = ref(100) // µF
const freq = ref(50) // Hz
const Vm = ref(220) // Peak voltage (V)

// ── Derived physics ─────────────────────────────────────────────────────────
const omega = computed(() => 2 * Math.PI * freq.value)

const XL = computed(() => omega.value * (L.value / 1000)) // L in H
const XC = computed(() => 1 / (omega.value * (C.value / 1e6))) // C in F
const X = computed(() => XL.value - XC.value)
const Z = computed(() => Math.sqrt(R.value ** 2 + X.value ** 2))

const Im = computed(() => Vm.value / Z.value)
const phi = computed(() => Math.atan2(X.value, R.value)) // radians
const phiDeg = computed(() => (phi.value * 180) / Math.PI)

const f0 = computed(() => 1 / (2 * Math.PI * Math.sqrt((L.value / 1000) * (C.value / 1e6))))
const VR_peak = computed(() => Im.value * R.value)
const VL_peak = computed(() => Im.value * XL.value)
const VC_peak = computed(() => Im.value * XC.value)

const pfCos = computed(() => Math.cos(phi.value))
const resonant = computed(() => Math.abs(f0.value - freq.value) < 0.5)

// ── Waveform canvas ──────────────────────────────────────────────────────────
const waveCanvas = ref<HTMLCanvasElement | null>(null)
const phasorCanvas = ref<HTMLCanvasElement | null>(null)

let animTime = 0

function drawWaveform(ctx: CanvasRenderingContext2D, w: number, h: number, t: number) {
  ctx.clearRect(0, 0, w, h)

  // Background
  ctx.fillStyle = '#0F1923'
  ctx.fillRect(0, 0, w, h)

  // Grid lines
  ctx.strokeStyle = '#253549'
  ctx.lineWidth = 1
  for (let i = 1; i < 4; i++) {
    const y = (h / 4) * i
    ctx.beginPath()
    ctx.moveTo(0, y)
    ctx.lineTo(w, y)
    ctx.stroke()
  }
  // Vertical grid
  for (let i = 1; i < 8; i++) {
    const x = (w / 8) * i
    ctx.beginPath()
    ctx.moveTo(x, 0)
    ctx.lineTo(x, h)
    ctx.stroke()
  }

  // Center axis
  ctx.strokeStyle = '#4A6180'
  ctx.lineWidth = 1.5
  ctx.beginPath()
  ctx.moveTo(0, h / 2)
  ctx.lineTo(w, h / 2)
  ctx.stroke()

  const cycles = 2
  const points = 400
  const omg = omega.value
  const phi_ = phi.value
  const vm = Vm.value
  const im = Im.value
  const amp = h * 0.4

  function drawWave(color: string, fn: (idx: number) => number, peak: number) {
    ctx.beginPath()
    ctx.strokeStyle = color
    ctx.lineWidth = 2
    for (let i = 0; i <= points; i++) {
      const tLocal = ((i / points) * cycles) / freq.value + t
      const val = fn(tLocal) / peak
      const x = (i / points) * w
      const y = h / 2 - val * amp
      if (i === 0) {
        ctx.moveTo(x, y)
      } else {
        ctx.lineTo(x, y)
      }
    }
    ctx.stroke()
  }

  // Voltage source
  drawWave('#FF6B4A', (tl) => vm * Math.sin(omg * tl), vm)
  // Current
  drawWave('#38BDF8', (tl) => im * Math.sin(omg * tl - phi_), im)
  // VR
  drawWave('#FFB830', (tl) => im * R.value * Math.sin(omg * tl - phi_), VR_peak.value || 1)

  // Legend
  const legends = [
    { color: '#FF6B4A', label: 'V(t)' },
    { color: '#38BDF8', label: 'I(t)' },
    { color: '#FFB830', label: 'VR(t)' },
  ]
  legends.forEach((l, i) => {
    ctx.fillStyle = l.color
    ctx.fillRect(10, 10 + i * 18, 20, 3)
    ctx.fillStyle = '#8B9DB5'
    ctx.font = '11px Be Vietnam Pro, sans-serif'
    ctx.fillText(l.label, 36, 16 + i * 18)
  })
}

function drawPhasor(ctx: CanvasRenderingContext2D, w: number, h: number) {
  ctx.clearRect(0, 0, w, h)
  ctx.fillStyle = '#0F1923'
  ctx.fillRect(0, 0, w, h)

  const cx = w / 2,
    cy = h / 2
  const maxR = Math.min(w, h) * 0.38

  // Grid circles
  ctx.strokeStyle = '#253549'
  ctx.lineWidth = 1
  for (let r = maxR * 0.33; r <= maxR; r += maxR * 0.33) {
    ctx.beginPath()
    ctx.arc(cx, cy, r, 0, 2 * Math.PI)
    ctx.stroke()
  }
  // Axes
  ctx.strokeStyle = '#4A6180'
  ctx.lineWidth = 1.5
  ctx.beginPath()
  ctx.moveTo(0, cy)
  ctx.lineTo(w, cy)
  ctx.stroke()
  ctx.beginPath()
  ctx.moveTo(cx, 0)
  ctx.lineTo(cx, h)
  ctx.stroke()

  // Axis labels
  ctx.fillStyle = '#4A6180'
  ctx.font = '11px Be Vietnam Pro, sans-serif'
  ctx.fillText('+Re', w - 28, cy - 6)
  ctx.fillText('+Im', cx + 6, 14)

  const scale = maxR / (Vm.value || 1)

  function arrow(
    color: string,
    angle: number, // radians from +X (Re axis), angle measured counterclockwise
    mag: number,
    label: string,
  ) {
    const ex = cx + mag * scale * Math.cos(angle)
    const ey = cy - mag * scale * Math.sin(angle)
    ctx.beginPath()
    ctx.strokeStyle = color
    ctx.lineWidth = 2
    ctx.moveTo(cx, cy)
    ctx.lineTo(ex, ey)
    ctx.stroke()

    // Arrowhead
    const headLen = 8
    const headAngle = Math.atan2(cy - ey, ex - cx)
    ctx.beginPath()
    ctx.fillStyle = color
    ctx.moveTo(ex, ey)
    ctx.lineTo(ex - headLen * Math.cos(headAngle - 0.4), ey + headLen * Math.sin(headAngle - 0.4))
    ctx.lineTo(ex - headLen * Math.cos(headAngle + 0.4), ey + headLen * Math.sin(headAngle + 0.4))
    ctx.closePath()
    ctx.fill()

    // Label
    ctx.fillStyle = color
    ctx.font = 'bold 11px Be Vietnam Pro, sans-serif'
    ctx.fillText(label, ex + 5, ey - 5)
  }

  // V source (reference, angle = 0 → along Re axis)
  arrow('#FF6B4A', 0, Vm.value, 'Vs')
  // VR (in phase with I, lags V by phi)
  const phiV = -phi.value // phasor from +Re
  arrow('#FFB830', phiV, VR_peak.value, 'VR')
  // VL leads I by 90°
  arrow('#38BDF8', phiV + Math.PI / 2, VL_peak.value, 'VL')
  // VC lags I by 90°
  arrow('#A78BFA', phiV - Math.PI / 2, VC_peak.value, 'VC')

  // Phase angle arc
  if (Math.abs(phi.value) > 0.05) {
    ctx.beginPath()
    ctx.strokeStyle = '#8B9DB5'
    ctx.lineWidth = 1
    ctx.setLineDash([4, 3])
    const arcR = maxR * 0.18
    ctx.arc(cx, cy, arcR, -phiV, 0, phi.value > 0)
    ctx.stroke()
    ctx.setLineDash([])
    ctx.fillStyle = '#8B9DB5'
    ctx.font = '10px Be Vietnam Pro, sans-serif'
    const midA = -phiV / 2
    ctx.fillText('φ', cx + arcR * 1.3 * Math.cos(midA), cy - arcR * 1.3 * Math.sin(midA))
  }
}

const { pause, resume } = useRafFn(
  () => {
    animTime += 1 / 60

    const wc = waveCanvas.value
    if (wc) {
      const ctx = wc.getContext('2d')
      if (ctx) drawWaveform(ctx, wc.width, wc.height, animTime)
    }

    const pc = phasorCanvas.value
    if (pc) {
      const ctx = pc.getContext('2d')
      if (ctx) drawPhasor(ctx, pc.width, pc.height)
    }
  },
  { immediate: false },
)

function resizeCanvases() {
  const wc = waveCanvas.value
  if (wc) {
    wc.width = wc.offsetWidth * window.devicePixelRatio
    wc.height = wc.offsetHeight * window.devicePixelRatio
    const ctx = wc.getContext('2d')
    if (ctx) ctx.scale(window.devicePixelRatio, window.devicePixelRatio)
  }
  const pc = phasorCanvas.value
  if (pc) {
    pc.width = pc.offsetWidth * window.devicePixelRatio
    pc.height = pc.offsetHeight * window.devicePixelRatio
    const ctx = pc.getContext('2d')
    if (ctx) ctx.scale(window.devicePixelRatio, window.devicePixelRatio)
  }
}

onMounted(() => {
  resizeCanvases()
  resume()
  window.addEventListener('resize', resizeCanvases)
})

onUnmounted(() => {
  pause()
  window.removeEventListener('resize', resizeCanvases)
})

// Reset to resonance
function setResonance() {
  // f0 = 1/(2π√LC), set freq to f0 based on current L, C
  const f = 1 / (2 * Math.PI * Math.sqrt((L.value / 1000) * (C.value / 1e6)))
  freq.value = Math.round(Math.min(Math.max(f, 1), 2000))
}

function fmt(n: number, d = 2) {
  return isFinite(n) ? n.toFixed(d) : '∞'
}
</script>

<template>
  <div class="min-h-screen bg-bg-deep text-text-primary font-body">
    <!-- Header -->
    <header
      class="border-b border-border-default px-6 py-4 flex items-center justify-between animate-fade-up"
    >
      <div class="flex items-center gap-3">
        <RouterLink
          to="/"
          class="flex items-center gap-1.5 text-text-secondary text-sm transition hover:text-accent-coral"
        >
          <Icon icon="lucide:arrow-left" class="size-4" />
          Trang chủ
        </RouterLink>
        <span class="text-border-default">│</span>
        <span class="text-accent-coral font-display text-sm tracking-widest">//</span>
        <h1 class="font-display text-lg font-semibold text-text-primary">Mô phỏng mạch RLC</h1>
      </div>
      <div class="flex items-center gap-2">
        <span
          v-if="resonant"
          class="bg-accent-coral/20 text-accent-coral border border-accent-coral/30 text-xs font-display tracking-wider px-2 py-1 animate-pulse"
        >
          CỘNG HƯỞNG
        </span>
        <button
          class="flex items-center gap-1.5 border border-border-default bg-bg-surface px-3 py-1.5 text-xs text-text-secondary transition hover:border-accent-amber hover:text-accent-amber"
          @click="setResonance"
        >
          <Icon icon="lucide:zap" class="size-3.5" />
          Đặt f₀
        </button>
      </div>
    </header>

    <div class="mx-auto max-w-7xl px-4 py-6 space-y-6">
      <!-- ── Row 1: Circuit diagram + Parameters ── -->
      <div class="grid grid-cols-1 lg:grid-cols-3 gap-4 animate-fade-up animate-delay-1">
        <!-- Circuit diagram (SVG) -->
        <div class="lg:col-span-2 border border-border-default bg-bg-surface p-4">
          <p class="text-accent-sky font-display text-xs tracking-widest mb-3">// SƠ ĐỒ MẠCH</p>
          <svg viewBox="0 0 640 200" class="w-full" style="max-height: 200px">
            <!-- Wire segments -->
            <line x1="40" y1="100" x2="100" y2="100" stroke="#253549" stroke-width="2" />
            <line x1="200" y1="100" x2="260" y2="100" stroke="#253549" stroke-width="2" />
            <line x1="360" y1="100" x2="420" y2="100" stroke="#253549" stroke-width="2" />
            <line x1="520" y1="100" x2="600" y2="100" stroke="#253549" stroke-width="2" />
            <!-- Top wire -->
            <line x1="600" y1="100" x2="600" y2="40" stroke="#253549" stroke-width="2" />
            <line x1="600" y1="40" x2="40" y2="40" stroke="#253549" stroke-width="2" />
            <line x1="40" y1="40" x2="40" y2="100" stroke="#253549" stroke-width="2" />

            <!-- AC source -->
            <circle cx="40" cy="100" r="20" fill="none" stroke="#FF6B4A" stroke-width="2" />
            <path
              d="M32,100 Q36,88 40,100 Q44,112 48,100"
              fill="none"
              stroke="#FF6B4A"
              stroke-width="1.5"
            />
            <text
              x="40"
              y="140"
              text-anchor="middle"
              fill="#FF6B4A"
              font-size="11"
              font-family="Be Vietnam Pro"
            >
              Vs=220V
            </text>

            <!-- Resistor (zigzag) -->
            <polyline
              points="100,100 110,88 120,112 130,88 140,112 150,88 160,112 170,88 180,100 200,100"
              fill="none"
              stroke="#FFB830"
              stroke-width="2"
            />
            <text
              x="150"
              y="76"
              text-anchor="middle"
              fill="#FFB830"
              font-size="11"
              font-family="Be Vietnam Pro"
            >
              R={{ R }}Ω
            </text>

            <!-- Inductor (bumps) -->
            <path
              d="M260,100 Q270,82 280,100 Q290,82 300,100 Q310,82 320,100 Q330,82 340,100 L360,100"
              fill="none"
              stroke="#38BDF8"
              stroke-width="2"
            />
            <text
              x="310"
              y="76"
              text-anchor="middle"
              fill="#38BDF8"
              font-size="11"
              font-family="Be Vietnam Pro"
            >
              L={{ L }}mH
            </text>

            <!-- Capacitor (two parallel lines) -->
            <line x1="420" y1="82" x2="420" y2="118" stroke="#A78BFA" stroke-width="2.5" />
            <line x1="430" y1="82" x2="430" y2="118" stroke="#A78BFA" stroke-width="2.5" />
            <line x1="420" y1="100" x2="420" y2="100" stroke="none" />
            <!-- wires to cap -->
            <line x1="420" y1="100" x2="420" y2="100" stroke="#253549" stroke-width="2" />
            <line x1="460" y1="82" x2="460" y2="118" stroke="#A78BFA" stroke-width="2.5" />
            <line x1="450" y1="82" x2="450" y2="118" stroke="#A78BFA" stroke-width="2.5" />
            <line x1="420" y1="100" x2="450" y2="100" stroke="#253549" stroke-width="2" />
            <line x1="460" y1="100" x2="520" y2="100" stroke="#253549" stroke-width="2" />
            <text
              x="470"
              y="76"
              text-anchor="middle"
              fill="#A78BFA"
              font-size="11"
              font-family="Be Vietnam Pro"
            >
              C={{ C }}µF
            </text>

            <!-- Dots at nodes -->
            <circle cx="40" cy="40" r="3" fill="#4A6180" />
            <circle cx="600" cy="40" r="3" fill="#4A6180" />

            <!-- Labels: + - -->
            <text x="58" y="94" fill="#FF6B4A" font-size="13" font-family="Be Vietnam Pro">+</text>
            <text x="14" y="94" fill="#FF6B4A" font-size="13" font-family="Be Vietnam Pro">−</text>

            <!-- Frequency annotation -->
            <text
              x="580"
              y="80"
              fill="#8B9DB5"
              font-size="10"
              font-family="Be Vietnam Pro"
              text-anchor="end"
            >
              f={{ freq }}Hz
            </text>
          </svg>
        </div>

        <!-- Parameters panel -->
        <div class="border border-border-default bg-bg-surface p-5 space-y-4">
          <p class="text-accent-amber font-display text-xs tracking-widest">// THÔNG SỐ</p>

          <!-- R -->
          <div class="flex items-center justify-between gap-2">
            <span class="text-text-secondary font-display text-xs shrink-0">Điện trở R</span>
            <div class="flex items-center gap-1">
              <input
                v-model.number="R"
                type="number"
                min="1"
                max="99999"
                step="1"
                class="rlc-number-input w-24 text-accent-amber"
              />
              <span class="text-text-dim text-xs font-display">Ω</span>
            </div>
          </div>

          <!-- L -->
          <div class="flex items-center justify-between gap-2">
            <span class="text-text-secondary font-display text-xs shrink-0">Cuộn cảm L</span>
            <div class="flex items-center gap-1">
              <input
                v-model.number="L"
                type="number"
                min="1"
                max="99999"
                step="1"
                class="rlc-number-input w-24 text-accent-sky"
              />
              <span class="text-text-dim text-xs font-display">mH</span>
            </div>
          </div>

          <!-- C -->
          <div class="flex items-center justify-between gap-2">
            <span class="text-text-secondary font-display text-xs shrink-0">Tụ điện C</span>
            <div class="flex items-center gap-1">
              <input
                v-model.number="C"
                type="number"
                min="1"
                max="99999"
                step="1"
                class="rlc-number-input w-24"
                style="color: #a78bfa"
              />
              <span class="text-text-dim text-xs font-display">µF</span>
            </div>
          </div>

          <!-- Frequency -->
          <div class="flex items-center justify-between gap-2">
            <span class="text-text-secondary font-display text-xs shrink-0">Tần số f</span>
            <div class="flex items-center gap-1">
              <input
                v-model.number="freq"
                type="number"
                min="1"
                max="99999"
                step="1"
                class="rlc-number-input w-24 text-accent-coral"
              />
              <span class="text-text-dim text-xs font-display">Hz</span>
            </div>
          </div>

          <!-- Vm -->
          <div class="flex items-center justify-between gap-2">
            <span class="text-text-secondary font-display text-xs shrink-0">Điện áp đỉnh Vm</span>
            <div class="flex items-center gap-1">
              <input
                v-model.number="Vm"
                type="number"
                min="1"
                max="9999"
                step="1"
                class="rlc-number-input w-24 text-text-primary"
              />
              <span class="text-text-dim text-xs font-display">V</span>
            </div>
          </div>

          <!-- Resonance indicator -->
          <div class="border-t border-border-default pt-3">
            <p class="text-xs text-text-dim font-display">Tần số cộng hưởng</p>
            <p class="text-lg font-display font-bold text-accent-coral mt-0.5">
              {{ fmt(f0, 1) }} Hz
            </p>
          </div>
        </div>
      </div>

      <!-- ── Row 2: Waveform + Phasor ── -->
      <div class="grid grid-cols-1 lg:grid-cols-3 gap-4 animate-fade-up animate-delay-2">
        <!-- Waveform -->
        <div class="lg:col-span-2 border border-border-default bg-bg-surface p-4">
          <p class="text-accent-coral font-display text-xs tracking-widest mb-3">// DẠNG SÓNG</p>
          <canvas ref="waveCanvas" class="w-full" style="height: 200px; display: block" />
        </div>

        <!-- Phasor -->
        <div class="border border-border-default bg-bg-surface p-4">
          <p class="text-accent-sky font-display text-xs tracking-widest mb-3">// GIẢN ĐỒ PHASOR</p>
          <canvas ref="phasorCanvas" class="w-full" style="height: 200px; display: block" />
          <!-- Phasor legend -->
          <div class="mt-2 grid grid-cols-2 gap-x-4 gap-y-1 text-xs">
            <div class="flex items-center gap-1.5">
              <span class="w-3 h-0.5 inline-block" style="background: #ff6b4a"></span>
              <span class="text-text-dim">Vs</span>
            </div>
            <div class="flex items-center gap-1.5">
              <span class="w-3 h-0.5 inline-block" style="background: #ffb830"></span>
              <span class="text-text-dim">VR</span>
            </div>
            <div class="flex items-center gap-1.5">
              <span class="w-3 h-0.5 inline-block" style="background: #38bdf8"></span>
              <span class="text-text-dim">VL</span>
            </div>
            <div class="flex items-center gap-1.5">
              <span class="w-3 h-0.5 inline-block" style="background: #a78bfa"></span>
              <span class="text-text-dim">VC</span>
            </div>
          </div>
        </div>
      </div>

      <!-- ── Row 3: Calculated values ── -->
      <div
        class="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-6 gap-3 animate-fade-up animate-delay-3"
      >
        <div
          v-for="(item, i) in [
            { label: 'Tổng trở Z', value: fmt(Z) + ' Ω', color: 'text-accent-coral' },
            { label: 'Dòng điện Im', value: fmt(Im, 3) + ' A', color: 'text-accent-sky' },
            { label: 'Góc pha φ', value: fmt(phiDeg, 1) + '°', color: 'text-accent-amber' },
            { label: 'ZL', value: fmt(XL) + ' Ω', color: 'text-accent-sky' },
            { label: 'ZC', value: fmt(XC) + ' Ω', color: 'text-purple-400' },
            { label: 'Hệ số công suất', value: fmt(pfCos, 3), color: 'text-text-primary' },
          ]"
          :key="i"
          class="border border-border-default bg-bg-surface p-4 transition-all duration-300 hover:border-accent-coral hover:bg-bg-elevated"
        >
          <p class="text-xs text-text-dim font-display mb-1">{{ item.label }}</p>
          <p class="text-xl font-display font-bold" :class="item.color">{{ item.value }}</p>
        </div>
      </div>

      <!-- ── Row 4: Voltage drops ── -->
      <div class="grid grid-cols-1 sm:grid-cols-3 gap-3 animate-fade-up animate-delay-4">
        <div
          v-for="(v, i) in [
            {
              label: 'Sụt áp VR (đỉnh)',
              val: fmt(VR_peak) + ' V',
              bar: VR_peak / Vm,
              color: '#FFB830',
            },
            {
              label: 'Sụt áp VL (đỉnh)',
              val: fmt(VL_peak) + ' V',
              bar: Math.min(VL_peak / Vm, 2),
              color: '#38BDF8',
            },
            {
              label: 'Sụt áp VC (đỉnh)',
              val: fmt(VC_peak) + ' V',
              bar: Math.min(VC_peak / Vm, 2),
              color: '#A78BFA',
            },
          ]"
          :key="i"
          class="border border-border-default bg-bg-surface p-4"
        >
          <p class="text-xs text-text-dim font-display mb-1">{{ v.label }}</p>
          <p class="text-2xl font-display font-bold mb-3" :style="{ color: v.color }">
            {{ v.val }}
          </p>
          <div class="h-1.5 bg-bg-deep w-full overflow-hidden">
            <div
              class="h-full transition-all duration-300"
              :style="{
                width: Math.min(v.bar * 100, 200) + '%',
                background: v.color,
                maxWidth: '100%',
              }"
            />
          </div>
        </div>
      </div>

      <!-- ── Footer ── -->
      <div
        class="border-t border-border-default pt-4 flex items-center justify-between text-xs text-text-dim animate-fade-up animate-delay-5"
      >
        <span class="font-display tracking-wide">RLC SERIES CIRCUIT SIMULATOR</span>
        <span>Nguồn AC lý tưởng · Mạch nối tiếp · Toàn tính toán thực</span>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Number inputs */
.rlc-number-input {
  background: #0f1923;
  border: 1px solid #253549;
  outline: none;
  padding: 3px 6px;
  font-size: 13px;
  font-family: 'Anybody', sans-serif;
  font-weight: 600;
  text-align: right;
  transition: border-color 0.2s;
}
.rlc-number-input:focus {
  border-color: #ff6b4a;
}
/* Remove spin arrows for cleaner look */
.rlc-number-input::-webkit-outer-spin-button,
.rlc-number-input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  appearance: none;
  margin: 0;
}
.rlc-number-input[type='number'] {
  appearance: textfield;
  -moz-appearance: textfield;
}
</style>
