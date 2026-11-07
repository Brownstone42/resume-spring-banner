<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
const badge = ref(null), cardAssembly = ref(null)
const taps = ref(0), active = ref(false), reduced = ref(false), dragging = ref(false)
const ribbon = ref([])
const clamp = (n, min, max) => Math.max(min, Math.min(max, n))
// Local CSS pixels: fixed ceiling anchor and a freely moving attachment point.
// Slower motion scale, preserving the same resting strap length.
const anchorX = 145, ropeLength = 132, gravity = 400, stiffness = 50
// Absorb stretch along the strap, but retain sideways pendulum momentum.
const radialDamping = 6, swingDamping = .65
const restY = ropeLength + gravity / stiffness
let x = anchorX, y = restY, vx = 0, vy = 0, rotation = -7, spin = 0
let bend = 0, bendSpeed = 0, drag = null, suppressClick = false
let frame = 0, last = 0, tapTimer, media
function drawRibbon() {
  const dx = x - anchorX, distance = Math.hypot(dx, y)
  const slack = Math.max(0, ropeLength - distance)
  const a = rotation * Math.PI / 180
  const handle = Math.min(32, Math.max(12, distance * .23))
  // Gravity makes loose fabric sag; stretched fabric straightens toward the card.
  const c1 = { x: anchorX + dx * .32 + bend + slack * .35, y: y * .32 + slack * .9 }
  const c2 = { x: x + Math.sin(a) * handle, y: y - Math.cos(a) * handle + slack * .45 }
  const points = []
  for (let i = 0; i <= 40; i++) {
    const t = i / 40, u = 1 - t
    const px = u*u*u*anchorX + 3*u*u*t*c1.x + 3*u*t*t*c2.x + t*t*t*x
    const py = 3*u*u*t*c1.y + 3*u*t*t*c2.y + t*t*t*y
    const tx = 3*u*u*(c1.x-anchorX) + 6*u*t*(c2.x-c1.x) + 3*t*t*(x-c2.x)
    const ty = 3*u*u*c1.y + 6*u*t*(c2.y-c1.y) + 3*t*t*(y-c2.y)
    const length = Math.max(.001, Math.hypot(tx, ty))
    points.push({ x: px, y: py, nx: ty / length, ny: -tx / length })
  }
  ribbon.value = points.slice(0, -1).map((p, i) => {
    const q = points[i+1]
    const edge = (v, sign, width = 11.5) => (v.x + sign*v.nx*width).toFixed(2)+','+(v.y + sign*v.ny*width).toFixed(2)
    return { shape: [edge(p,1),edge(q,1),edge(q,-1),edge(p,-1)].join(' '),
      seam: 'M'+edge(p,-1,8.5)+'L'+edge(q,-1,8.5), color: 'hsl(87 15% '+(25 + 2*Math.cos(i/40*Math.PI))+'%)' }
  })
}
function paint() {
  drawRibbon()
  if (badge.value) badge.value.style.transform = 'none'
  if (cardAssembly.value) cardAssembly.value.style.transform = 'translate('+ (x-anchorX).toFixed(3)+'px,'+(y-restY).toFixed(3)+'px) rotate('+rotation.toFixed(3)+'deg)'
}
function resetMotion() {
  x = anchorX; y = restY; vx = vy = spin = bend = bendSpeed = 0; rotation = -7
}
function startDrag(e) {
  if (!e.isPrimary || e.button !== 0 || drag) return
  const scene = badge.value.parentElement
  const scale = scene.getBoundingClientRect().width / scene.offsetWidth
  suppressClick = false
  drag = { id:e.pointerId, element:e.currentTarget, scale, startX:e.clientX, startY:e.clientY,
    originX:x, originY:y, sampleX:x, sampleY:y, time:e.timeStamp, moved:false }
  vx = vy = spin = 0
  e.currentTarget.setPointerCapture(e.pointerId)
  dragging.value = true
}
function dragMove(e) {
  if (!drag || e.pointerId !== drag.id) return
  const dx = e.clientX-drag.startX, dy = e.clientY-drag.startY
  if (!drag.moved && Math.hypot(dx,dy) < 5) return
  drag.moved = true
  x = drag.originX + dx/drag.scale; y = drag.originY + dy/drag.scale
  const dt = (e.timeStamp-drag.time)/1000
  if (dt > 0) {
    const blend = 1-Math.exp(-dt*24)
    vx += (clamp((x-drag.sampleX)/dt,-1600,1600)-vx)*blend
    vy += (clamp((y-drag.sampleY)/dt,-1600,1600)-vy)*blend
  }
  drag.sampleX=x; drag.sampleY=y; drag.time=e.timeStamp
  paint()
}
function endDrag(e) {
  if (!drag || (e?.pointerId != null && e.pointerId !== drag.id)) return
  const previous = drag
  suppressClick = previous.moved || e?.type === 'pointercancel'
  if (!e || e.type !== 'pointerup' || e.timeStamp-previous.time > 100) vx=vy=0
  drag=null; dragging.value=false
  if (previous.element.hasPointerCapture(previous.id)) previous.element.releasePointerCapture(previous.id)
  if (reduced.value) { resetMotion(); paint() }
}
function blur() { endDrag() }
function tap(event) {
  if (event.detail !== 0 && suppressClick) { suppressClick=false; return }
  taps.value++
  if (!reduced.value) {
    const box=cardAssembly.value.querySelector('.id-card').getBoundingClientRect()
    const side=event.detail ? (event.clientX < box.left+box.width/2 ? 1 : -1) : (taps.value%2 ? 1 : -1)
    vx=clamp(vx+side*260,-1000,1000); spin+=side*35
  }
  active.value=true; clearTimeout(tapTimer)
  tapTimer=setTimeout(()=>{ active.value=false },450)
}
function animate(time) {
  const elapsed=Math.min((time-(last||time))/1000,.04); last=time
  const steps=Math.max(1,Math.ceil(elapsed/(1/180))), dt=elapsed/steps
  if (drag && time-drag.time > 60) { const decay=Math.exp(-elapsed*18); vx*=decay;vy*=decay }
  for (let i=0;i<steps;i++) {
    if (!drag && !reduced.value) {
      const dx=x-anchorX, distance=Math.max(.001,Math.hypot(dx,y))
      const nx=dx/distance, ny=y/distance
      const stretch=Math.max(0,distance-ropeLength)
      const radialVelocity=vx*nx+vy*ny
      // A tether pulls only: inside its resting length the badge falls freely.
      const tension=stretch>0 ? Math.max(0,stiffness*stretch+radialDamping*radialVelocity) : 0
      vx+=(-tension*nx-swingDamping*vx)*dt
      vy+=(gravity-tension*ny-swingDamping*vy)*dt
      x+=vx*dt; y+=vy*dt
      const desiredRotation=-7+clamp(-Math.atan2(dx,Math.max(30,y))*180/Math.PI*.45-vx*.012,-25,25)
      spin+=((desiredRotation-rotation)*32-spin*10)*dt; rotation+=spin*dt
      if (Math.hypot(x-anchorX,y-restY)<.08 && Math.hypot(vx,vy)<.15 && Math.abs(rotation+7)<.08) resetMotion()
    }
    const desiredBend=reduced.value ? 0 : clamp(-vx*.025,-12,12)
    bendSpeed+=((desiredBend-bend)*75-bendSpeed*18)*dt; bend+=bendSpeed*dt
  }
  paint()
  if (!reduced.value) frame=requestAnimationFrame(animate)
}
function preference() {
  endDrag(); reduced.value=media.matches; cancelAnimationFrame(frame)
  resetMotion(); last=0; frame=requestAnimationFrame(animate)
}
onMounted(()=>{
  media=window.matchMedia('(prefers-reduced-motion: reduce)'); preference()
  media.addEventListener('change',preference); window.addEventListener('blur',blur)
})
onUnmounted(()=>{
  endDrag();cancelAnimationFrame(frame);clearTimeout(tapTimer)
  media?.removeEventListener('change',preference);window.removeEventListener('blur',blur)
})
</script>

<template>
  <div class="page-shell">
    <header class="flex items-center justify-between gap-4 border-b border-stone-300/70 px-6 py-6 md:px-12">
      <a class="wordmark" href="#home" aria-label="Anawat home">anawat<span>®</span></a>
      <div class="hidden items-center gap-2 text-xs sm:flex"><span class="status-dot"></span> OPEN TO POSSIBILITIES</div>
      <a class="nav-link" href="#about">ABOUT THIS DEMO <span>↗</span></a>
    </header>

    <main id="home">
      <section class="hero relative overflow-hidden" aria-labelledby="hero-title">
        <div class="hero-grid" aria-hidden="true"></div>
        <div class="hero-copy relative z-10">
          <div class="eyebrow flex items-center gap-3"><span class="tiny-star">✳</span> CREATIVE DEVELOPER <span class="muted">/ BANGKOK, TH</span></div>
          <h1 id="hero-title">A little logic.<br>A lot of <span class="play-word">play.<svg viewBox="0 0 360 25" fill="none" aria-hidden="true"><path d="M5 16C86 2 215 1 350 12M49 23C161 12 264 11 323 17" stroke="currentColor" stroke-width="3" stroke-linecap="round"/></svg></span></h1>
          <p class="intro">I build thoughtful digital experiences.<br>Clean interfaces, curious details, and things<br class="desktop-break"> that feel good to interact with.</p>
          <div class="flex flex-wrap items-center gap-5 pt-8">
            <a href="#about" class="primary-button">Explore the demo <span>↘</span></a>
            <span class="small-note">Made to be played with.</span>
          </div>
        </div>

        <div class="badge-scene">
          <div class="anchor" aria-hidden="true"></div>
          <div ref="badge" class="pendulum">
            <svg class="fabric-strap" viewBox="0 0 290 160" aria-hidden="true"><g v-for="(strip, i) in ribbon" :key="i"><polygon :points="strip.shape" :fill="strip.color" :stroke="strip.color" stroke-width="0.6"/><path :d="strip.seam" stroke="#bac3a2" stroke-opacity=".45" stroke-width=".7" stroke-dasharray="1.5 1.5"/></g></svg>
            <div ref="cardAssembly" class="card-assembly">
            <div class="metal-clip" aria-hidden="true"></div>
            <button class="id-card" :class="{ tapped: active, dragging }" @pointerdown="startDrag" @pointermove="dragMove" @pointerup="endDrag" @pointercancel="endDrag" @lostpointercapture="endDrag" @click="tap" @dragstart.prevent aria-label="ลากป้ายแล้วปล่อยให้แกว่ง หรือคลิกสะกิด กด Enter หรือ Space เพื่อสะกิดได้เช่นกัน">
              <div class="card-top"><span>HELLO, I'M</span><span class="card-star">✳</span></div>
              <div class="avatar-art" aria-hidden="true">
                <svg viewBox="0 0 240 170" fill="none"><circle cx="120" cy="85" r="74" fill="#e4dbbd"/><path d="M49 176c1-44 25-65 71-65s70 21 71 65" fill="#252c29"/><path d="M98 101h44v28c-15 12-29 12-44 0z" fill="#e8a474"/><ellipse cx="120" cy="77" rx="37" ry="44" fill="#efb98d"/><path d="M82 76c-8-40 10-60 40-60 35 0 43 26 35 56l-14-29c-11 13-29 15-51 12z" fill="#272e29"/><path d="M97 73h18v13H97zm28 0h18v13h-18z" stroke="#272e29" stroke-width="4"/><path d="M115 78h10m-15 21c7 6 15 6 22-1" stroke="#272e29" stroke-width="3" stroke-linecap="round"/><path d="m91 132 29 27 29-27" stroke="#f3f1e9" stroke-width="2"/><path d="M120 159v17" stroke="#f3f1e9" stroke-width="2"/></svg>
                <span class="avatar-label">BUILD. BREAK. REPEAT.</span>
              </div>
              <div class="card-name">ANAWAT<span>Creative developer</span></div>
              <div class="card-bottom"><span class="barcode" aria-hidden="true"></span><span>GOOD HUMAN<br>NO. 001 — DEMO</span></div>
            </button>
            </div>
          </div>
          <div class="tap-hint"><svg viewBox="0 0 85 55" fill="none" aria-hidden="true"><path d="M3 47c34 4 63-4 63-34m-12 8 12-15 11 13" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg><span>Grab, drag & let go.</span></div>
        </div>

        <div class="hero-bottom flex items-center justify-between gap-4">
          <span>DESIGN MINDED. DETAIL OBSESSED.</span>
          <span class="interaction-status" aria-live="polite">{{ taps ? 'NICE TAP. × ' + taps : 'CLICK OR DRAG THE BADGE.' }}</span>
        </div>
      </section>
      <section id="about" class="about-strip grid gap-8 border-t border-stone-300/70 px-6 py-9 md:grid-cols-[1fr_1.6fr_1fr] md:px-12">
        <div><div class="section-number">01 / THE PLAYGROUND</div><h2>Small details.<br>Real personality.</h2></div>
        <div class="about-copy"><p>ตัวอย่าง Resume Banner ด้วย Vue + Tailwind CSS</p><p>จับป้ายลากได้ทั้งซ้าย ขวา ขึ้น และลง ยกขึ้นให้สายหย่อนแล้วปล่อย สายจะดึงป้ายกลับมาแกว่งและค่อย ๆ หยุด หรือคลิกเบา ๆ เพื่อสะกิด ลองลากด้วยนิ้วบนมือถือได้ด้วย</p><span class="demo-label">ชื่อ ตำแหน่ง และภาพบุคคลเป็นเนื้อหาตัวอย่าง</span></div>
        <div class="flex flex-wrap content-start gap-2 md:justify-end"><span class="tech-chip">Vue 3</span><span class="tech-chip">Tailwind CSS</span><span class="tech-chip">Spring physics</span><span class="tech-chip">No 3D engine</span></div>
      </section>
    </main>
    <footer class="flex justify-between gap-4 border-t border-stone-300/70 px-6 py-5 text-xs text-stone-500 md:px-12"><span>ANAWAT — INTERACTIVE RESUME CONCEPT</span><span>Built with a little curiosity. ↗</span></footer>
  </div>
</template>
