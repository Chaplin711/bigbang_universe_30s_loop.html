<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Big Bang → Cosmic Web (30s Loop)</title>
  <style>
    html, body { margin:0; height:100%; background:#000; overflow:hidden; font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial; }
    canvas { width:100%; height:100%; display:block; }
    .hud {
      position: fixed; left: 16px; top: 16px; z-index: 2;
      color: #eaeaea; background: rgba(0,0,0,0.45);
      padding: 10px 12px; border: 1px solid rgba(255,255,255,0.15);
      border-radius: 12px; backdrop-filter: blur(6px);
      font-size: 12px; line-height: 1.35; max-width: 420px;
      user-select: none;
    }
    .hud b { font-weight: 700; }
    .row { margin-top: 8px; display:flex; gap:10px; flex-wrap: wrap; align-items:center; }
    button {
      border: 1px solid rgba(255,255,255,0.2);
      background: rgba(255,255,255,0.06);
      color: #fff; padding: 6px 10px; border-radius: 10px;
      cursor: pointer;
      font-size: 12px;
    }
    button:hover { background: rgba(255,255,255,0.12); }
    .pill {
      font-size: 12px; padding: 2px 8px; border-radius: 999px;
      border: 1px solid rgba(255,255,255,0.18); opacity: 0.92;
    }
  </style>
</head>
<body>
  <canvas id="c"></canvas>

  <div class="hud" id="hud">
    <div><b>Big Bang → Cosmic Web (30s Perfect Loop)</b></div>
    <div style="opacity:.9">Keys: <b>H</b>=hide HUD, <b>L</b>=toggle labels, <b>Space</b>=pause</div>
    <div class="row">
      <span class="pill" id="phase">Phase</span>
      <span class="pill" id="time">t</span>
    </div>
    <div style="opacity:.85; margin-top:6px">
      Presentation tip: record exactly 30s. Start/end are black for seamless looping.
    </div>
  </div>

<script>
(() => {
  const canvas = document.getElementById("c");
  const ctx = canvas.getContext("2d", { alpha: false });
  const phaseEl = document.getElementById("phase");
  const timeEl = document.getElementById("time");
  const hud = document.getElementById("hud");

  const DPR = Math.max(1, Math.min(2, window.devicePixelRatio || 1));
  let W=0,H=0, CX=0, CY=0, S=0;

  function resize(){
    W = Math.floor(innerWidth * DPR);
    H = Math.floor(innerHeight * DPR);
    canvas.width = W; canvas.height = H;
    CX = W*0.5; CY = H*0.5;
    S = Math.min(W,H);
  }
  addEventListener("resize", resize);
  resize();

  // --- deterministic RNG (seeded) ---
  let seed = 133742069;
  function rand01(){
    seed = (1664525 * seed + 1013904223) >>> 0;
    return seed / 4294967296;
  }
  function rand(a,b){ return a + (b-a) * rand01(); }
  function clamp(x,a,b){ return Math.max(a, Math.min(b,x)); }

  // --- 30s loop ---
  const LOOP_SEC = 30;
  const LOOP_MS = LOOP_SEC * 1000;
  const FADE_SEC = 0.75; // black-in + black-out for seamless loop
  let paused = false;
  let start = performance.now();

  // toggles
  let showLabels = true;

  // --- Dark matter filaments (cosmic web): static polylines + gentle flow field ---
  // Build a few filaments as wavy polylines in "comoving" space ([-1..1])
  const filaments = [];
  function buildFilaments(){
    filaments.length = 0;
    seed = 987654321;
    const k = 7;
    for (let i=0;i<k;i++){
      const points = [];
      const x0 = rand(-1,1), y0 = rand(-1,1);
      const dx = rand(-0.6,0.6), dy = rand(-0.6,0.6);
      const len = rand(1.2, 2.3);
      const steps = 18;
      for (let j=0;j<=steps;j++){
        const u = j/steps;
        // wavy line
        const wob = 0.10*Math.sin((u*6 + rand(-0.5,0.5))*Math.PI);
        const px = x0 + dx*(u*len) + wob*Math.cos((i+1)*2.1);
        const py = y0 + dy*(u*len) + wob*Math.sin((i+1)*1.7);
        points.push({x:px, y:py});
      }
      filaments.push({ points, phase: rand(0, Math.PI*2) });
    }
  }
  buildFilaments();

  // Flow field pulls matter weakly toward nearest filament segment (visual metaphor)
  function filamentPull(xc, yc, t01){
    // xc,yc in comoving coords
    let best = null;
    let bestD2 = Infinity;

    for (const f of filaments){
      const pts = f.points;
      for (let i=0;i<pts.length-1;i++){
        const ax=pts[i].x, ay=pts[i].y;
        const bx=pts[i+1].x, by=pts[i+1].y;
        const abx=bx-ax, aby=by-ay;
        const apx=xc-ax, apy=yc-ay;
        const ab2 = abx*abx + aby*aby + 1e-9;
        let u = (apx*abx + apy*aby)/ab2;
        u = clamp(u,0,1);
        const px = ax + u*abx, py = ay + u*aby;
        const dx = px-xc, dy = py-yc;
        const d2 = dx*dx + dy*dy;
        if (d2 < bestD2){
          bestD2 = d2;
          best = {dx, dy};
        }
      }
    }

    if (!best) return {fx:0, fy:0};
    const d = Math.sqrt(bestD2) + 0.02;
    // pull is stronger later (structure formation)
    const strength = 0.018 * smoothstep(0.25, 0.95, t01);
    return { fx: (best.dx/d)*strength, fy: (best.dy/d)*strength };
  }

  function smoothstep(a,b,x){
    const t = clamp((x-a)/(b-a), 0, 1);
    return t*t*(3-2*t);
  }

  // --- Expansion factor: very rapid early, then slower ---
  // We don't need to end where we started because we fade to black.
  function expansionFactor(t01){
    // t01 = 0..1 across 30s
    // early inflation burst then gradual Hubble-like expansion
    const inflation = Math.exp(-t01*12.0);      // huge early, quickly fades
    const late = 1.0 + 2.6 * smoothstep(0.06, 1.0, t01);
    return late * (1.0 + 0.9*inflation);
  }

  // --- Fade envelope for perfect loop (black at start and end) ---
  function loopFade(tSec){
    // 0..1 opacity for content
    const inFade = smoothstep(0, FADE_SEC, tSec);
    const outFade = 1 - smoothstep(LOOP_SEC-FADE_SEC, LOOP_SEC, tSec);
    return clamp(inFade*outFade, 0, 1);
  }

  // --- Bodies ---
  const plasma = [];
  const quasars = [];
  const pulsars = [];
  const comets = [];
  const galaxies = [];
  const systems = [];

  function makePlasma(n=1400){
    plasma.length=0;
    seed = 42424242;
    for(let i=0;i<n;i++){
      // very dense near origin in comoving space
      const r = Math.pow(rand01(), 6) * 0.08;
      const a = rand(0, Math.PI*2);
      plasma.push({
        cx: Math.cos(a)*r,
        cy: Math.sin(a)*r,
        // outward direction
        vx: Math.cos(a) * rand(0.03, 0.10),
        vy: Math.sin(a) * rand(0.03, 0.10),
        // temperature / brightness
        hot: rand(0.7, 1.0)
      });
    }
  }

  function makeGalaxies(){
    galaxies.length = 0;
    seed = 111999222;
    // Named galaxies
    galaxies.push(makeSpiralGalaxy("Milky Way", -0.35,  0.10,  0.55,  0.016, 0.85));
    galaxies.push(makeSpiralGalaxy("Andromeda",  0.38, -0.08, -0.45,  0.014, 0.90));
    // Additional galaxies (mix)
    const extra = 10;
    for(let i=0;i<extra;i++){
      const spiral = rand01() < 0.65;
      const x = rand(-0.9,0.9);
      const y = rand(-0.7,0.7);
      const rot = rand(-0.7,0.7);
      const rotSpd = rand(0.008, 0.020) * (rand01()<0.5?-1:1);
      const size = rand(0.55, 1.0);
      galaxies.push(spiral ? makeSpiralGalaxy("", x, y, rot, rotSpd, size) : makeEllipticalGalaxy("", x, y, rot, rotSpd, size));
    }
  }

  function makeSpiralGalaxy(name, x, y, rot0, rotSpd, size){
    return { kind:"spiral", name, x, y, rot0, rotSpd, size, spawn: 9.5, bright: 0.9 };
  }
  function makeEllipticalGalaxy(name, x, y, rot0, rotSpd, size){
    return { kind:"elliptical", name, x, y, rot0, rotSpd, size, spawn: 11.0, bright: 0.75 };
  }

  function makeQuasars(n=6){
    quasars.length=0;
    seed = 999333111;
    // Ensure a few appear early-ish
    for(let i=0;i<n;i++){
      quasars.push({
        x: rand(-0.7,0.7),
        y: rand(-0.5,0.5),
        spawn: rand(3.0, 9.0),
        pulse: rand(2.0, 6.0)
      });
    }
  }

  function makePulsars(n=10){
    pulsars.length=0;
    seed = 555777999;
    for(let i=0;i<n;i++){
      pulsars.push({
        x: rand(-0.85,0.85),
        y: rand(-0.65,0.65),
        spawn: rand(14.0, 22.0),
        spin: rand(8.0, 18.0)
      });
    }
  }

  function makeSystems(){
    systems.length=0;
    seed = 20202020;

    // Named Sun system (visual metaphor, not scale-accurate)
    systems.push({
      name: "Sun",
      x: -0.08, y: 0.22,
      spawn: 20.5,
      planets: [
        { r: 0.030, w:  4.0 },
        { r: 0.045, w:  3.1 },
        { r: 0.065, w:  2.4 },
        { r: 0.090, w:  1.8 },
      ]
    });

    // Some extra unnamed systems
    const extra = 6;
    for(let i=0;i<extra;i++){
      const planets = [];
      const pcount = 3 + Math.floor(rand01()*4);
      for(let j=0;j<pcount;j++){
        planets.push({ r: rand(0.02,0.10)*(1+j*0.45), w: rand(1.2, 5.5) });
      }
      systems.push({
        name: "",
        x: rand(-0.85,0.85),
        y: rand(-0.65,0.65),
        spawn: rand(19.0, 25.0),
        planets
      });
    }
  }

  function makeComets(n=16){
    comets.length=0;
    seed = 80808080;
    for(let i=0;i<n;i++){
      comets.push({
        // wide orbits around some random focal point
        fx: rand(-0.6,0.6),
        fy: rand(-0.4,0.4),
        a: rand(0.20, 0.65),     // semi-major in comoving
        b: rand(0.08, 0.25),     // semi-minor
        phase: rand(0, Math.PI*2),
        speed: rand(0.6, 1.8),   // cycles over loop
        spawn: rand(10.0, 29.0)
      });
    }
  }

  function init(){
    makePlasma();
    makeGalaxies();
    makeQuasars();
    makePulsars();
    makeSystems();
    makeComets();
  }
  init();

  // --- Background stars ---
  const stars = [];
  function buildStars(){
    stars.length=0;
    seed = 314159265;
    for(let i=0;i<260;i++){
      stars.push({ x: rand01(), y: rand01(), r: (rand01()**2)*1.4, a: rand(0.15,0.55) });
    }
  }
  buildStars();

  function drawStars(contentAlpha){
    for(const s of stars){
      ctx.fillStyle = `rgba(255,255,255,${s.a*contentAlpha})`;
      ctx.beginPath();
      ctx.arc(s.x*W, s.y*H, s.r*DPR, 0, Math.PI*2);
      ctx.fill();
    }
  }

  // --- Colors ---
  function plasmaColor(hot, alpha){
    // hot: 1 => white/yellow, cooler => blue-ish
    const t = clamp(hot,0,1);
    const r = Math.round(120 + (255-120)*t);
    const g = Math.round(160 + (235-160)*t);
    const b = Math.round(255 + (190-255)*t);
    return `rgba(${r},${g},${b},${alpha})`;
  }

  // --- Helpers: comoving to screen with expansion + filament drift ---
  function toScreen(xc, yc, t01){
    const a = expansionFactor(t01);
    // apply filament pull as a drift (later)
    const fp = filamentPull(xc, yc, t01);
    const drift = 0.55 * smoothstep(0.18, 1.0, t01);
    const x = CX + (xc + fp.fx*drift) * (S*0.45) * a;
    const y = CY + (yc + fp.fy*drift) * (S*0.45) * a;
    return {x, y};
  }

  // --- Phase labeling ---
  function phaseName(tSec){
    if (tSec < 4.0)  return "Big Bang / Plasma Era";
    if (tSec < 10.0) return "Quasar Era";
    if (tSec < 16.5) return "Galaxy Formation";
    if (tSec < 21.0) return "Stellar Remnants (Pulsars)";
    if (tSec < 26.0) return "Solar Systems & Planets";
    return "Cosmic Web (Present Day)";
  }

  // --- Draw dark matter filaments ---
  function drawFilaments(tSec, contentAlpha){
    const t01 = tSec / LOOP_SEC;
    const a = expansionFactor(t01);
    // faint purple-blue
    ctx.lineWidth = 1.2 * DPR;
    for (const f of filaments){
      const wob = 0.015 * Math.sin(f.phase + t01*2*Math.PI);
      ctx.beginPath();
      for (let i=0;i<f.points.length;i++){
        const p = f.points[i];
        const px = p.x + wob*Math.sin(i*0.9);
        const py = p.y + wob*Math.cos(i*0.8);
        const s = toScreen(px, py, t01);
        if (i===0) ctx.moveTo(s.x, s.y);
        else ctx.lineTo(s.x, s.y);
      }
      ctx.strokeStyle = `rgba(120,140,255,${0.13*contentAlpha})`;
      ctx.stroke();
    }

    // faint nodes on intersections/ends
    for (const f of filaments){
      const ends = [f.points[0], f.points[f.points.length-1]];
      for (const e of ends){
        const s = toScreen(e.x, e.y, t01);
        ctx.fillStyle = `rgba(160,180,255,${0.12*contentAlpha})`;
        ctx.beginPath();
        ctx.arc(s.x, s.y, 3.0*DPR, 0, Math.PI*2);
        ctx.fill();
      }
    }
  }

  // --- Draw spiral galaxy (rotating) ---
  function drawSpiralGalaxy(g, tSec, contentAlpha){
    const t01 = tSec / LOOP_SEC;
    if (tSec < g.spawn) return;

    const rot = g.rot0 + g.rotSpd * (tSec - g.spawn);
    const center = toScreen(g.x, g.y, t01);

    // Core
    const coreR = (8 * g.size) * DPR;
    ctx.fillStyle = `rgba(255,255,255,${0.18*contentAlpha})`;
    ctx.beginPath(); ctx.arc(center.x, center.y, coreR, 0, Math.PI*2); ctx.fill();

    // Arms (points along logarithmic-ish spiral)
    const arms = 2;
    const pts = 120;
    const maxR = (85 * g.size) * DPR;
    ctx.lineWidth = 1.0 * DPR;

    for (let arm=0; arm<arms; arm++){
      ctx.beginPath();
      for (let i=0;i<pts;i++){
        const u = i/(pts-1);
        const r = u*u * maxR;
        const theta = rot + arm*Math.PI + u*5.6; // swirl
        const x = center.x + Math.cos(theta)*r;
        const y = center.y + Math.sin(theta)*r;
        if (i===0) ctx.moveTo(x,y); else ctx.lineTo(x,y);
      }
      ctx.strokeStyle = `rgba(140,200,255,${0.10*contentAlpha})`;
      ctx.stroke();
    }

    // Star field within galaxy disk
    const starCount = 260;
    seed = (Math.floor((g.x+2)*1e6) ^ Math.floor((g.y+2)*1e6) ^ 0x9e3779b9) >>> 0;
    for (let i=0;i<starCount;i++){
      const u = rand01();
      const r = (u**1.8) * maxR;
      const theta = rot + rand(0, Math.PI*2) + (r/maxR)*2.8;
      const x = center.x + Math.cos(theta)*r + rand(-0.6,0.6)*DPR;
      const y = center.y + Math.sin(theta)*r + rand(-0.6,0.6)*DPR;
      const a = 0.10 + 0.25*(1-u);
      ctx.fillStyle = `rgba(200,220,255,${a*0.55*contentAlpha})`;
      ctx.beginPath();
      ctx.arc(x,y, (0.8 + 1.2*(1-u))*DPR, 0, Math.PI*2);
      ctx.fill();
    }

    // Label
    if (showLabels && g.name){
      drawLabel(g.name, center.x, center.y - maxR*0.55, contentAlpha);
    }
  }

  function drawEllipticalGalaxy(g, tSec, contentAlpha){
    const t01 = tSec / LOOP_SEC;
    if (tSec < g.spawn) return;

    const rot = g.rot0 + g.rotSpd * (tSec - g.spawn);
    const center = toScreen(g.x, g.y, t01);
    const rx = (70 * g.size) * DPR;
    const ry = (45 * g.size) * DPR;

    ctx.save();
    ctx.translate(center.x, center.y);
    ctx.rotate(rot);
    // glow
    const grad = ctx.createRadialGradient(0,0, 0, 0,0, rx);
    grad.addColorStop(0, `rgba(220,240,255,${0.12*contentAlpha})`);
    grad.addColorStop(1, `rgba(0,0,0,0)`);
    ctx.fillStyle = grad;
    ctx.beginPath();
    ctx.ellipse(0,0, rx, ry, 0, 0, Math.PI*2);
    ctx.fill();

    // stars
    seed = (Math.floor((g.x+3)*1e6) ^ Math.floor((g.y+3)*1e6) ^ 0x85ebca6b) >>> 0;
    for(let i=0;i<220;i++){
      const u = rand01();
      const a = rand(0, Math.PI*2);
      const r = (u**1.7);
      const x = Math.cos(a)*rx*r + rand(-0.8,0.8)*DPR;
      const y = Math.sin(a)*ry*r + rand(-0.8,0.8)*DPR;
      ctx.fillStyle = `rgba(230,240,255,${(0.08+0.22*(1-u))*contentAlpha})`;
      ctx.beginPath();
      ctx.arc(x,y, (0.8+1.2*(1-u))*DPR, 0, Math.PI*2);
      ctx.fill();
    }
    ctx.restore();
  }

  // --- Quasar (bright core + jets) ---
  function drawQuasar(q, tSec, contentAlpha){
    if (tSec < q.spawn) return;
    const t01 = tSec / LOOP_SEC;
    const pos = toScreen(q.x, q.y, t01);
    const pulse = 0.55 + 0.45*Math.sin((tSec-q.spawn)*q.pulse);
    const r = (6 + 10*pulse) * DPR;

    // core
    ctx.fillStyle = `rgba(220,180,255,${0.22*contentAlpha})`;
    ctx.beginPath(); ctx.arc(pos.x,pos.y,r,0,Math.PI*2); ctx.fill();

    // jets
    const jetLen = (40 + 60*pulse) * DPR;
    ctx.strokeStyle = `rgba(190,140,255,${0.15*contentAlpha})`;
    ctx.lineWidth = 2.0*DPR;
    ctx.beginPath();
    ctx.moveTo(pos.x-jetLen, pos.y);
    ctx.lineTo(pos.x+jetLen, pos.y);
    ctx.stroke();

    if (showLabels && tSec < q.spawn + 6.5){
      drawLabel("Quasar", pos.x, pos.y - 24*DPR, contentAlpha*0.9);
    }
  }

  // --- Pulsar (beaming) ---
  function drawPulsar(p, tSec, contentAlpha){
    if (tSec < p.spawn) return;
    const t01 = tSec / LOOP_SEC;
    const pos = toScreen(p.x, p.y, t01);
    const ang = (tSec - p.spawn) * p.spin;

    ctx.fillStyle = `rgba(160,255,180,${0.22*contentAlpha})`;
    ctx.beginPath(); ctx.arc(pos.x,pos.y, 4.0*DPR, 0, Math.PI*2); ctx.fill();

    const beam = 70*DPR;
    ctx.strokeStyle = `rgba(120,255,160,${0.12*contentAlpha})`;
    ctx.lineWidth = 1.8*DPR;
    ctx.beginPath();
    ctx.moveTo(pos.x + Math.cos(ang)*beam, pos.y + Math.sin(ang)*beam);
    ctx.lineTo(pos.x - Math.cos(ang)*beam, pos.y - Math.sin(ang)*beam);
    ctx.stroke();

    if (showLabels && tSec < p.spawn + 5.5){
      drawLabel("Pulsar", pos.x, pos.y - 22*DPR, contentAlpha*0.85);
    }
  }

  // --- Solar system (star + orbiting planets) ---
  function drawSystem(sys, tSec, contentAlpha){
    if (tSec < sys.spawn) return;
    const t01 = tSec / LOOP_SEC;
    const pos = toScreen(sys.x, sys.y, t01);

    // star
    ctx.fillStyle = `rgba(255,235,170,${0.22*contentAlpha})`;
    ctx.beginPath(); ctx.arc(pos.x,pos.y, 5.2*DPR, 0, Math.PI*2); ctx.fill();

    // orbits + planets
    ctx.lineWidth = 1.0*DPR;
    for (let i=0;i<sys.planets.length;i++){
      const pl = sys.planets[i];
      const rr = (pl.r * S * 0.45) * 0.22 * DPR; // orbit radius in pixels-ish
      const ang = (tSec - sys.spawn) * (pl.w * 0.9) + i*1.7;

      // orbit
      ctx.strokeStyle = `rgba(200,200,200,${0.07*contentAlpha})`;
      ctx.beginPath(); ctx.arc(pos.x,pos.y, rr, 0, Math.PI*2); ctx.stroke();

      // planet
      const px = pos.x + Math.cos(ang)*rr;
      const py = pos.y + Math.sin(ang)*rr;
      ctx.fillStyle = `rgba(180,210,255,${0.16*contentAlpha})`;
      ctx.beginPath(); ctx.arc(px,py, (1.8+0.5*(i%2))*DPR, 0, Math.PI*2); ctx.fill();
    }

    if (showLabels && sys.name){
      drawLabel(sys.name, pos.x, pos.y - 22*DPR, contentAlpha);
    }
  }

  // --- Comets (elliptical paths + tail) ---
  function drawComet(c, tSec, contentAlpha){
    if (tSec < c.spawn) return;
    const t01 = tSec / LOOP_SEC;

    // move along ellipse using time (wraps automatically)
    const u = ( (tSec - c.spawn) * c.speed / LOOP_SEC ) % 1.0;
    const ang = (u * Math.PI*2) + c.phase;

    const xc = c.fx + Math.cos(ang)*c.a;
    const yc = c.fy + Math.sin(ang)*c.b;

    const pos = toScreen(xc, yc, t01);
    const tail = 28*DPR;

    ctx.strokeStyle = `rgba(255,255,255,${0.10*contentAlpha})`;
    ctx.lineWidth = 1.2*DPR;
    ctx.beginPath();
    ctx.moveTo(pos.x, pos.y);
    ctx.lineTo(pos.x - tail, pos.y - tail*0.35);
    ctx.stroke();

    ctx.fillStyle = `rgba(255,255,255,${0.14*contentAlpha})`;
    ctx.beginPath();
    ctx.arc(pos.x, pos.y, 2.2*DPR, 0, Math.PI*2);
    ctx.fill();

    if (showLabels && tSec < c.spawn + 4.0){
      drawLabel("Comet", pos.x + 14*DPR, pos.y - 10*DPR, contentAlpha*0.8);
    }
  }

  // --- Plasma particles (early) ---
  function drawPlasma(tSec, contentAlpha){
    const t01 = tSec / LOOP_SEC;

    // Plasma fades out after ~12s
    const live = 1 - smoothstep(8.0, 14.0, tSec);
    if (live <= 0.001) return;

    const a = expansionFactor(t01);
    const cooling = 1 - smoothstep(0.02, 0.45, t01); // hot early
    const baseSize = (2.2 - 1.1*smoothstep(0.0, 0.5, t01)) * DPR;

    for (const p of plasma){
      // comoving motion outward
      const xc = p.cx + p.vx * (tSec*0.12);
      const yc = p.cy + p.vy * (tSec*0.12);

      // filament drift (later)
      const fp = filamentPull(xc, yc, t01);
      const drift = 0.6 * smoothstep(0.22, 1.0, t01);

      const x = CX + (xc + fp.fx*drift) * (S*0.45) * a;
      const y = CY + (yc + fp.fy*drift) * (S*0.45) * a;

      const hot = clamp(p.hot*cooling + 0.08, 0, 1);
      const alpha = (0.18 + 0.32*hot) * live * contentAlpha;
      ctx.fillStyle = plasmaColor(hot, alpha);

      ctx.beginPath();
      ctx.arc(x, y, baseSize*(0.55 + 0.55*p.hot), 0, Math.PI*2);
      ctx.fill();
    }
  }

  function drawLabel(text, x, y, alpha){
    ctx.font = `${12*DPR}px system-ui, -apple-system, Segoe UI, Roboto, Arial`;
    ctx.fillStyle = `rgba(255,255,255,${0.85*alpha})`;
    ctx.strokeStyle = `rgba(0,0,0,${0.55*alpha})`;
    ctx.lineWidth = 3*DPR;
    ctx.strokeText(text, x, y);
    ctx.fillText(text, x, y);
  }

  // --- Main draw loop ---
  function frame(now){
    if (!paused) {
      // keep deterministic loop time
    } else {
      // freeze time by not advancing start (we'll just reuse last computed t)
    }

    const elapsed = now - start;
    const tMs = elapsed % LOOP_MS;
    const tSec = tMs / 1000;
    const contentAlpha = loopFade(tSec);

    // UI
    phaseEl.textContent = phaseName(tSec);
    timeEl.textContent = `t = ${tSec.toFixed(2)} / ${LOOP_SEC.toFixed(0)}s`;

    // Clear to black
    ctx.fillStyle = "#000";
    ctx.fillRect(0,0,W,H);

    // Stars (fade with content)
    drawStars(contentAlpha);

    // Dark matter filaments (cosmic web)
    drawFilaments(tSec, contentAlpha);

    // Plasma (early)
    drawPlasma(tSec, contentAlpha);

    // Quasars
    for (const q of quasars) drawQuasar(q, tSec, contentAlpha);

    // Galaxies (spiral rotate!)
    for (const g of galaxies) {
      if (g.kind === "spiral") drawSpiralGalaxy(g, tSec, contentAlpha);
      else drawEllipticalGalaxy(g, tSec, contentAlpha);
    }

    // Pulsars
    for (const p of pulsars) drawPulsar(p, tSec, contentAlpha);

    // Systems (Sun + planets)
    for (const s of systems) drawSystem(s, tSec, contentAlpha);

    // Comets
    for (const c of comets) drawComet(c, tSec, contentAlpha);

    // subtle vignette
    const vign = ctx.createRadialGradient(CX,CY, S*0.2, CX,CY, S*0.7);
    vign.addColorStop(0, "rgba(0,0,0,0)");
    vign.addColorStop(1, `rgba(0,0,0,${0.65*(1-contentAlpha) + 0.18})`);
    ctx.fillStyle = vign;
    ctx.fillRect(0,0,W,H);

    requestAnimationFrame(frame);
  }

  requestAnimationFrame(frame);

  // Controls
  addEventListener("keydown", (e) => {
    if (e.code === "Space") {
      e.preventDefault();
      paused = !paused;
      if (!paused) {
        // reset start so the loop resumes smoothly from current position
        // (we keep the current tSec by shifting start)
        const now = performance.now();
        const elapsed = now - start;
        const tMs = elapsed % LOOP_MS;
        start = now - tMs;
      }
    }
    if (e.key.toLowerCase() === "h") hud.style.display = (hud.style.display === "none" ? "block" : "none");
    if (e.key.toLowerCase() === "l") showLabels = !showLabels;
  });

})();
</script>
</body>
</html>
