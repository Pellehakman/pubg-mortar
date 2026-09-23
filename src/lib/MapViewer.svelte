<script>
  import { untrack } from 'svelte';

  let {
    maps = [],
    spreadMeters = 5, // ungefärlig spridning för ett nedslag
    maxRange = 696, // mortarens maxräckvidd
    arcShape = 0.53 // hur snabbt banan planar ut med avståndet (1 = konstant fart)
  } = $props();

  const G = 9.81;
  const MAX_SCALE = 6;
  const CLICK_SLOP = 5;
  const HIT_RADIUS = 16;
  const H_LIMIT = 200;

  let container = $state(null);
  let containerW = $state(0);
  let containerH = $state(0);

  let mapId = $state(maps[0]?.id);
  const map = $derived(maps.find((m) => m.id === mapId) ?? maps[0]);
  const mpp = $derived(map ? map.meters / map.pixels : 1);

  let scale = $state(1);
  let tx = $state(0);
  let ty = $state(0);

  let origin = $state(null); // avfyrningsplats
  let target = $state(null); // nedslag
  let cursor = $state(null);
  let panning = $state(false);
  let sheetOpen = $state(false); // bottenpanelens läge på mobil

  let deltaH = $state(0); // negativt = målet ligger lägre

  let ready = false;
  const pointers = new Map();
  let moved = 0;
  let last = { x: 0, y: 0 };
  let pinch = null;
  let multitouch = false;

  const minScale = $derived(
    map && containerW && containerH
      ? Math.max(0.02, Math.min(containerW, containerH) / map.pixels)
      : 0.05
  );

  const toScreen = (p) => (p ? { x: p.x * scale + tx, y: p.y * scale + ty } : null);
  const sOrigin = $derived(toScreen(origin));
  const sTarget = $derived(toScreen(target));
  const sCursor = $derived(toScreen(cursor));

  const spreadPx = $derived(Math.max(5, (spreadMeters / mpp) * scale));

  // === Ballistik =========================================================
  //
  // Mortaren väljer den höga banlösningen. Pipvinkeln för inställning S:
  //
  //     θ = 90° − ½·arcsin( (S / R_max)^arcShape )
  //
  // och farten väljs så att plan räckvidd blir exakt S — plana skott stämmer
  // därför alltid, oavsett parametrar. arcShape = 1 ger konstant fart.
  //
  // Kalibrerat mot: 403 m + 130 m höjd → 486, och 503 m + 130 m höjd → 612.

  const ratioFor = (setting) => Math.pow(Math.min(1, Math.max(0, setting / maxRange)), arcShape);
  const angleFor = (setting) => Math.PI / 2 - 0.5 * Math.asin(ratioFor(setting));
  const speedFor = (setting) => (G * setting) / Math.max(1e-9, ratioFor(setting));

  function heightAt(x, setting) {
    const theta = angleFor(setting);
    const c = Math.cos(theta);
    return x * Math.tan(theta) - (G * x * x) / (2 * speedFor(setting) * c * c);
  }

  function impactAngle(x, setting) {
    const theta = angleFor(setting);
    const c = Math.cos(theta);
    return (Math.atan((G * x) / (speedFor(setting) * c * c) - Math.tan(theta)) * 180) / Math.PI;
  }

  /**
   * Höjden vid ett fast avstånd växer inte hela vägen upp till maxräckvidden —
   * den toppar någonstans och faller sedan. Gyllene-snitt-sök hittar toppen,
   * sedan bisekterar vi på den växande delen.
   */
  function dialFor(distance, dh) {
    if (distance <= 0) return null;

    let a = 0.5;
    let b = maxRange;
    for (let i = 0; i < 60; i++) {
      const m1 = a + (b - a) / 3;
      const m2 = b - (b - a) / 3;
      if (heightAt(distance, m1) < heightAt(distance, m2)) a = m1;
      else b = m2;
    }
    const peak = (a + b) / 2;
    if (heightAt(distance, peak) < dh) return null; // utom räckhåll

    let lo = 0.5;
    let hi = peak;
    for (let i = 0; i < 50; i++) {
      const mid = (lo + hi) / 2;
      if (heightAt(distance, mid) < dh) lo = mid;
      else hi = mid;
    }
    return (lo + hi) / 2;
  }

  function solve(a, b) {
    if (!a || !b) return null;
    const dx = b.x - a.x;
    const dy = b.y - a.y;
    const dist = Math.hypot(dx, dy) * mpp;
    const bearing = ((Math.atan2(dx, -dy) * 180) / Math.PI + 360) % 360;
    const dial = dialFor(dist, deltaH);
    if (dial === null)
      return { map: dist, bearing, dial: null, delta: 0, launch: null, impact: null };
    return {
      map: dist,
      bearing,
      dial,
      delta: dial - dist,
      launch: (angleFor(dial) * 180) / Math.PI,
      impact: impactAngle(dist, dial)
    };
  }

  const solution = $derived(solve(origin, target));
  const preview = $derived(origin && cursor && !panning ? solve(origin, cursor) : null);

  const overOrigin = $derived(
    !!(sOrigin && sCursor && Math.hypot(sCursor.x - sOrigin.x, sCursor.y - sOrigin.y) <= HIT_RADIUS)
  );

  // === Banprofil i 1:1 ===================================================

  const BAR_STEPS = [25, 50, 100, 200, 400];

  const profile = $derived.by(() => {
    if (!solution || solution.dial === null || solution.map < 1) return null;

    const D = solution.map;
    const S = solution.dial;
    const W = 252;
    const H = 150;
    const pad = 10;
    const foot = 18;

    const pts = [];
    let maxY = 0;
    let minY = Math.min(0, deltaH);
    for (let i = 0; i <= 96; i++) {
      const x = (D * i) / 96;
      const y = heightAt(x, S);
      pts.push([x, y]);
      if (y > maxY) maxY = y;
      if (y < minY) minY = y;
    }

    const worldH = Math.max(1, maxY - minY);
    const boxW = W - pad * 2;
    const boxH = H - pad * 2 - foot;
    const k = Math.min(boxW / D, boxH / worldH); // samma skala i höjd och längd

    const ox = (W - D * k) / 2;
    const oy = pad + (boxH - worldH * k) / 2;
    const px = (x) => ox + x * k;
    const py = (y) => oy + (maxY - y) * k;

    const barMeters = [...BAR_STEPS].reverse().find((m) => m * k <= boxW * 0.5) ?? BAR_STEPS[0];

    return {
      W,
      H,
      path: pts.map(([x, y]) => `${px(x).toFixed(1)},${py(y).toFixed(1)}`).join(' '),
      start: { x: px(0), y: py(0) },
      end: { x: px(D), y: py(deltaH) },
      apex: Math.round(maxY),
      bar: { meters: barMeters, width: barMeters * k, x: pad, y: H - 8 }
    };
  });

  // === Transform =========================================================

  const clampScale = (s) => Math.min(MAX_SCALE, Math.max(minScale, s));

  function clampTranslate() {
    const size = map.pixels * scale;
    tx = size <= containerW ? (containerW - size) / 2 : Math.min(0, Math.max(containerW - size, tx));
    ty = size <= containerH ? (containerH - size) / 2 : Math.min(0, Math.max(containerH - size, ty));
  }

  function zoomAt(px, py, factor) {
    const next = clampScale(scale * factor);
    const k = next / scale;
    tx = px - (px - tx) * k;
    ty = py - (py - ty) * k;
    scale = next;
    clampTranslate();
  }

  const zoomCenter = (f) => zoomAt(containerW / 2, containerH / 2, f);

  function resetView() {
    scale = minScale;
    clampTranslate();
  }

  function chooseMap(id) {
    if (id === mapId) return;
    mapId = id;
    origin = null;
    target = null;
    cursor = null;
    resetView();
  }

  function toImage(clientX, clientY) {
    const r = container.getBoundingClientRect();
    return { x: (clientX - r.left - tx) / scale, y: (clientY - r.top - ty) / scale };
  }

  // === Interaktion =======================================================

  function onPointerDown(e) {
    container.setPointerCapture(e.pointerId);
    pointers.set(e.pointerId, { x: e.clientX, y: e.clientY });

    if (pointers.size === 1) {
      panning = true;
      moved = 0;
      last = { x: e.clientX, y: e.clientY };
    } else if (pointers.size === 2) {
      multitouch = true;
      panning = false;
      pinch = pinchState();
    }
  }

  function onPointerMove(e) {
    if (!container) return;
    const r = container.getBoundingClientRect();
    cursor = toImage(e.clientX, e.clientY);

    if (!pointers.has(e.pointerId)) return;
    pointers.set(e.pointerId, { x: e.clientX, y: e.clientY });

    if (pointers.size === 1 && panning) {
      const dx = e.clientX - last.x;
      const dy = e.clientY - last.y;
      tx += dx;
      ty += dy;
      moved += Math.hypot(dx, dy);
      last = { x: e.clientX, y: e.clientY };
      clampTranslate();
    } else if (pointers.size === 2 && pinch) {
      const now = pinchState();
      zoomAt(pinch.mid.x - r.left, pinch.mid.y - r.top, now.dist / pinch.dist);
      tx += now.mid.x - pinch.mid.x;
      ty += now.mid.y - pinch.mid.y;
      clampTranslate();
      pinch = now;
    }
  }

  function onPointerUp(e) {
    pointers.delete(e.pointerId);
    if (pointers.size < 2) pinch = null;

    if (pointers.size === 0) {
      if (panning && !multitouch && moved < CLICK_SLOP) place(e.clientX, e.clientY);
      panning = false;
      multitouch = false;
      if (e.pointerType !== 'mouse') cursor = null; // ingen hovring på touch
    }
  }

  function pinchState() {
    const [a, b] = [...pointers.values()];
    return {
      dist: Math.hypot(b.x - a.x, b.y - a.y) || 1,
      mid: { x: (a.x + b.x) / 2, y: (a.y + b.y) / 2 }
    };
  }

  function place(clientX, clientY) {
    const r = container.getBoundingClientRect();
    const px = clientX - r.left;
    const py = clientY - r.top;

    if (sOrigin && Math.hypot(px - sOrigin.x, py - sOrigin.y) <= HIT_RADIUS) {
      clearAll();
      return;
    }

    const p = { x: (px - tx) / scale, y: (py - ty) / scale };
    if (p.x < 0 || p.y < 0 || p.x > map.pixels || p.y > map.pixels) return;

    if (!origin) origin = p;
    else target = p;
  }

  function clearAll() {
    origin = null;
    target = null;
  }

  function setDelta(raw) {
    const n = Number(raw);
    deltaH = Number.isFinite(n) ? Math.max(-H_LIMIT, Math.min(H_LIMIT, Math.round(n))) : 0;
  }

  function onKeyDown(e) {
    if (e.target instanceof HTMLInputElement) return;
    if (e.key === 'Escape') target = null;
    if (e.key === 'Backspace') {
      e.preventDefault();
      clearAll();
    }
    if (e.key === '+' || e.key === '=') zoomCenter(1.3);
    if (e.key === '-') zoomCenter(1 / 1.3);
    if (e.key === '0') resetView();
  }

  $effect(() => {
    const el = container;
    if (!el) return;
    const onWheel = (e) => {
      e.preventDefault();
      const d = e.deltaMode === 1 ? e.deltaY * 16 : e.deltaY;
      const r = el.getBoundingClientRect();
      zoomAt(e.clientX - r.left, e.clientY - r.top, Math.exp(-d * 0.0015));
    };
    el.addEventListener('wheel', onWheel, { passive: false });
    return () => el.removeEventListener('wheel', onWheel);
  });

  $effect(() => {
    containerW;
    containerH;
    untrack(() => {
      if (!ready && containerW > 0 && containerH > 0) {
        ready = true;
        resetView();
      } else if (ready) {
        clampTranslate();
      }
    });
  });

  // === Presentation ======================================================

  const fmt = (m) => (m < 1000 ? `${Math.round(m)} m` : `${(m / 1000).toFixed(2)} km`);
  const brg = (d) => `${String(Math.round(d) % 360).padStart(3, '0')}°`;
  const signed = (m) => `${m >= 0 ? '+' : '−'}${Math.round(Math.abs(m))} m`;

  const STEPS = [10, 25, 50, 100, 250, 500, 1000, 2000, 5000];
  const bar = $derived.by(() => {
    const mPerPx = mpp / scale;
    const meters = STEPS.find((s) => s >= 120 * mPerPx) ?? STEPS[STEPS.length - 1];
    return { meters, width: meters / mPerPx };
  });

  const hint = $derived(
    !origin
      ? 'Tryck för att sätta avfyrningsplats'
      : overOrigin
        ? 'Tryck på pricken för att ta bort den'
        : !target
          ? 'Tryck på ett mål'
          : 'Tryck på nästa mål — avfyrningsplatsen ligger kvar'
  );

  const slopeWord = $derived(deltaH === 0 ? 'plant' : deltaH < 0 ? 'nedför' : 'uppför');
</script>

<svelte:window onkeydown={onKeyDown} />

<div class="viewer">
  <div
    class="stage"
    class:panning
    class:hot={overOrigin}
    bind:this={container}
    bind:clientWidth={containerW}
    bind:clientHeight={containerH}
    onpointerdown={onPointerDown}
    onpointermove={onPointerMove}
    onpointerup={onPointerUp}
    onpointercancel={onPointerUp}
    onpointerleave={() => (cursor = null)}
  >
    {#if map}
      <img
        src={map.src}
        alt={map.label}
        width={map.pixels}
        height={map.pixels}
        draggable="false"
        style="transform: translate({tx}px, {ty}px) scale({scale});"
      />
    {/if}

    <svg class="overlay" aria-hidden="true">
      {#if sOrigin && sCursor && preview && !overOrigin}
        <line class="aim" x1={sOrigin.x} y1={sOrigin.y} x2={sCursor.x} y2={sCursor.y} />
      {/if}

      {#if sOrigin && sTarget}
        <line class="shot-shadow" x1={sOrigin.x} y1={sOrigin.y} x2={sTarget.x} y2={sTarget.y} />
        <line
          class="shot"
          class:over={solution?.dial === null}
          x1={sOrigin.x}
          y1={sOrigin.y}
          x2={sTarget.x}
          y2={sTarget.y}
        />
      {/if}

      {#if sTarget}
        <circle class="spread-shadow" cx={sTarget.x} cy={sTarget.y} r={spreadPx} />
        <circle class="spread" cx={sTarget.x} cy={sTarget.y} r={spreadPx} />
        <circle class="impact" cx={sTarget.x} cy={sTarget.y} r="2" />
      {/if}

      {#if sOrigin}
        <circle class="origin-shadow" cx={sOrigin.x} cy={sOrigin.y} r="5" />
        <circle class="origin" cx={sOrigin.x} cy={sOrigin.y} r="4" />
      {/if}
    </svg>

    {#if sOrigin && sTarget && solution}
      <div
        class="tag"
        class:over={solution.dial === null}
        style="left: {(sOrigin.x + sTarget.x) / 2}px; top: {(sOrigin.y + sTarget.y) / 2}px;"
      >
        <strong>{solution.dial === null ? 'utom räckhåll' : fmt(solution.dial)}</strong>
        <em>{brg(solution.bearing)}</em>
      </div>
    {/if}

    {#if preview && sCursor && !overOrigin}
      <div class="ghost" style="left: {sCursor.x}px; top: {sCursor.y}px;">
        {#if preview.dial === null}
          {fmt(preview.map)} · utom räckhåll
        {:else if deltaH !== 0}
          {fmt(preview.map)} → <b>{fmt(preview.dial)}</b> · {brg(preview.bearing)}
        {:else}
          {fmt(preview.map)} · {brg(preview.bearing)}
        {/if}
      </div>
    {/if}

    <div class="tools">
      <button onclick={() => zoomCenter(1.4)} aria-label="Zooma in">+</button>
      <button onclick={() => zoomCenter(1 / 1.4)} aria-label="Zooma ut">−</button>
      <button onclick={resetView} aria-label="Visa hela kartan">
        <svg viewBox="0 0 16 16" width="13" height="13" aria-hidden="true">
          <path
            d="M2 6V2h4M14 6V2h-4M2 10v4h4M14 10v4h-4"
            fill="none"
            stroke="currentColor"
            stroke-width="1.6"
          />
        </svg>
      </button>
      <button onclick={clearAll} disabled={!origin} aria-label="Nollställ punkter">×</button>
    </div>

    <div class="scalebar">
      <span class="rule" style="width: {bar.width}px"></span>
      <span>{bar.meters >= 1000 ? `${bar.meters / 1000} km` : `${bar.meters} m`}</span>
    </div>
  </div>

  <aside class="panel" class:open={sheetOpen}>
    <button
      class="grip"
      onclick={() => (sheetOpen = !sheetOpen)}
      aria-expanded={sheetOpen}
      aria-label={sheetOpen ? 'Dölj inställningar' : 'Visa inställningar'}
    ></button>

    <div class="lede">
      <h1>Eldledning</h1>
      <p>{hint}</p>
    </div>

    <div class="dial" class:live={!!solution?.dial} class:over={solution?.dial === null}>
      <span class="label">Ställ in</span>
      <p class="figure">
        {#if !solution}
          <span class="none">—</span>
        {:else if solution.dial === null}
          <span class="none">utom räckhåll</span>
        {:else}
          {Math.round(solution.dial)}<small>m</small>
        {/if}
      </p>
      <dl class="facts">
        <div><dt>Avstånd</dt><dd>{solution ? fmt(solution.map) : '—'}</dd></div>
        <div><dt>Bäring</dt><dd>{solution ? brg(solution.bearing) : '—'}</dd></div>
        <div>
          <dt>Höjd</dt>
          <dd class:accent={deltaH !== 0}>{deltaH === 0 ? '0 m' : signed(deltaH)}</dd>
        </div>
      </dl>
      {#if solution?.dial != null && deltaH !== 0}
        <p class="delta">Höjden ger {signed(solution.delta)} på inställningen</p>
      {/if}
    </div>

    <div class="body">
      <section>
        <div class="head">
          <h2>Karta</h2>
          {#if map}<span>{(map.meters / 1000).toFixed(0)} × {(map.meters / 1000).toFixed(0)} km</span>{/if}
        </div>
        <div class="pills" role="group" aria-label="Välj karta">
          {#each maps as m}
            <button class:on={m.id === mapId} onclick={() => chooseMap(m.id)}>{m.label}</button>
          {/each}
        </div>
      </section>

      <section>
        <div class="head">
          <h2>Höjdskillnad</h2>
          <span class="slope" class:zero={deltaH === 0}>{slopeWord}</span>
        </div>
        <div class="hrow">
          <input
            class="slider"
            type="range"
            min={-H_LIMIT}
            max={H_LIMIT}
            step="10"
            value={deltaH}
            oninput={(e) => setDelta(e.currentTarget.value)}
            aria-label="Höjdskillnad i meter, negativt när målet ligger lägre"
          />
          <input
            class="num"
            type="number"
            min={-H_LIMIT}
            max={H_LIMIT}
            step="1"
            value={deltaH}
            onchange={(e) => setDelta(e.currentTarget.value)}
            aria-label="Höjdskillnad, exakt värde"
          />
        </div>
        <div class="ticks">
          <span>−{H_LIMIT} nedför</span>
          <span>uppför +{H_LIMIT}</span>
        </div>
      </section>

      <section>
        <div class="head">
          <h2>Bana</h2>
          {#if profile}<span>topp {profile.apex} m</span>{/if}
        </div>
        {#if profile && solution}
          <svg viewBox="0 0 {profile.W} {profile.H}" class="curve">
            <line
              class="ground"
              x1={profile.start.x}
              y1={profile.start.y}
              x2={profile.end.x}
              y2={profile.end.y}
            />
            <polyline class="flight" points={profile.path} />
            <circle class="from" cx={profile.start.x} cy={profile.start.y} r="3" />
            <circle class="to" cx={profile.end.x} cy={profile.end.y} r="3.5" />
            <g class="ref">
              <line
                x1={profile.bar.x}
                y1={profile.bar.y}
                x2={profile.bar.x + profile.bar.width}
                y2={profile.bar.y}
              />
              <line
                x1={profile.bar.x}
                y1={profile.bar.y - 3}
                x2={profile.bar.x}
                y2={profile.bar.y + 3}
              />
              <line
                x1={profile.bar.x + profile.bar.width}
                y1={profile.bar.y - 3}
                x2={profile.bar.x + profile.bar.width}
                y2={profile.bar.y + 3}
              />
              <text x={profile.bar.x + profile.bar.width + 5} y={profile.bar.y + 3}>
                {profile.bar.meters} m
              </text>
            </g>
          </svg>
          <div class="angles">
            <span>Pipvinkel <b>{Math.round(solution.launch)}°</b></span>
            <span>Nedslag <b>{Math.round(solution.impact)}°</b></span>
          </div>
        {:else}
          <p class="empty">Sätt ut ett mål så ritas banan här</p>
        {/if}
      </section>
    </div>
  </aside>
</div>

<style>
  .viewer {
    --bg: #050f13;
    --surface: #0a1a20;
    --raise: #102a32;
    --edge: rgb(255 255 255 / 0.09);
    --ink: #eaf2f4;
    --dim: #8ba9b2;
    --faint: #5b7883;
    --signal: #ff8a1f;
    --own: #ffffff;
    --warn: #ff5c4d;
    --sans: ui-sans-serif, system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif;
    --mono: ui-monospace, 'SF Mono', 'JetBrains Mono', Menlo, Consolas, monospace;

    position: relative;
    display: grid;
    grid-template-columns: 1fr 292px;
    height: 100%;
    background: var(--bg);
    color: var(--ink);
    font-family: var(--sans);
    font-size: 14px;
    -webkit-font-smoothing: antialiased;
  }

  /* --- Karta ----------------------------------------------------------- */

  .stage {
    position: relative;
    overflow: hidden;
    touch-action: none;
    cursor: crosshair;
    contain: strict;
  }
  .stage.panning {
    cursor: grabbing;
  }
  .stage.hot {
    cursor: pointer;
  }

  img {
    position: absolute;
    top: 0;
    left: 0;
    transform-origin: 0 0;
    user-select: none;
    -webkit-user-drag: none;
    will-change: transform;
  }

  .overlay {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
  }

  .origin {
    fill: var(--own);
  }
  .origin-shadow {
    fill: rgb(0 0 0 / 0.6);
  }
  .spread {
    fill: rgb(255 138 31 / 0.16);
    stroke: var(--signal);
    stroke-width: 1.5;
  }
  .spread-shadow {
    fill: none;
    stroke: #000;
    stroke-width: 3.5;
    opacity: 0.5;
  }
  .impact {
    fill: var(--signal);
  }
  .shot-shadow {
    stroke: #000;
    stroke-width: 4;
    opacity: 0.5;
  }
  .shot {
    stroke: var(--signal);
    stroke-width: 1.5;
    stroke-dasharray: 8 5;
  }
  .shot.over {
    stroke: var(--warn);
  }
  .aim {
    stroke: var(--own);
    stroke-width: 1;
    stroke-dasharray: 2 6;
    opacity: 0.28;
  }

  .tag {
    position: absolute;
    transform: translate(-50%, -50%);
    display: flex;
    align-items: baseline;
    gap: 7px;
    padding: 4px 9px;
    border-radius: 999px;
    background: rgb(5 15 19 / 0.86);
    border: 1px solid rgb(255 138 31 / 0.6);
    backdrop-filter: blur(8px);
    font-family: var(--mono);
    font-size: 12px;
    font-variant-numeric: tabular-nums;
    white-space: nowrap;
    pointer-events: none;
  }
  .tag.over {
    border-color: var(--warn);
    color: var(--warn);
  }
  .tag strong {
    font-weight: 600;
  }
  .tag em {
    font-style: normal;
    font-size: 10px;
    color: var(--dim);
  }

  .ghost {
    position: absolute;
    transform: translate(14px, 14px);
    font-family: var(--mono);
    font-size: 10px;
    font-variant-numeric: tabular-nums;
    color: rgb(255 255 255 / 0.6);
    text-shadow: 0 0 4px #000, 0 0 2px #000;
    white-space: nowrap;
    pointer-events: none;
  }
  .ghost b {
    color: var(--signal);
    font-weight: 600;
  }

  .tools {
    position: absolute;
    top: 14px;
    right: 14px;
    display: flex;
    flex-direction: column;
    gap: 1px;
    overflow: hidden;
    border-radius: 10px;
    border: 1px solid var(--edge);
    background: rgb(5 15 19 / 0.7);
    backdrop-filter: blur(10px);
  }
  .tools button {
    width: 34px;
    height: 34px;
    display: grid;
    place-items: center;
    padding: 0;
    border: 0;
    border-radius: 0;
    background: transparent;
    color: var(--ink);
    font-size: 15px;
    line-height: 1;
    cursor: pointer;
  }
  .tools button + button {
    box-shadow: inset 0 1px 0 var(--edge);
  }
  .tools button:hover:not(:disabled) {
    background: rgb(255 255 255 / 0.08);
    color: var(--signal);
  }
  .tools button:disabled {
    opacity: 0.3;
    cursor: default;
  }

  .scalebar {
    position: absolute;
    left: 16px;
    bottom: 16px;
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: var(--mono);
    font-size: 11px;
    text-shadow: 0 0 5px #000;
    pointer-events: none;
  }
  .rule {
    height: 6px;
    border: 1px solid var(--ink);
    border-top: 0;
    transition: width 0.1s linear;
  }

  /* --- Panel ------------------------------------------------------------ */

  .panel {
    display: flex;
    flex-direction: column;
    min-height: 0;
    background: var(--surface);
    border-left: 1px solid var(--edge);
    overflow-y: auto;
    overscroll-behavior: contain;
  }

  .grip {
    display: none;
  }

  .lede {
    padding: 18px 18px 14px;
  }
  h1 {
    margin: 0 0 5px;
    font-size: 11px;
    font-weight: 600;
    letter-spacing: 0.16em;
    text-transform: uppercase;
    color: var(--dim);
  }
  .lede p {
    margin: 0;
    font-size: 12px;
    line-height: 1.5;
    color: var(--faint);
  }

  .dial {
    padding: 0 18px 16px;
    border-bottom: 1px solid var(--edge);
  }
  .label {
    font-size: 10px;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    color: var(--faint);
  }
  .figure {
    margin: 2px 0 12px;
    font-family: var(--mono);
    font-size: 42px;
    font-weight: 600;
    line-height: 1;
    letter-spacing: -0.02em;
    font-variant-numeric: tabular-nums;
    color: var(--faint);
  }
  .dial.live .figure {
    color: var(--signal);
  }
  .figure small {
    margin-left: 3px;
    font-size: 15px;
    font-weight: 500;
    color: var(--dim);
  }
  .figure .none {
    font-size: 19px;
    letter-spacing: 0;
  }
  .dial.over .figure .none {
    color: var(--warn);
  }

  .facts {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 8px;
    margin: 0;
  }
  .facts div {
    display: flex;
    flex-direction: column;
    gap: 3px;
    min-width: 0;
  }
  dt {
    font-size: 10px;
    color: var(--faint);
  }
  dd {
    margin: 0;
    font-family: var(--mono);
    font-size: 13px;
    font-variant-numeric: tabular-nums;
    color: var(--ink);
  }
  dd.accent {
    color: var(--signal);
  }

  .delta {
    margin: 12px 0 0;
    padding: 7px 9px;
    border-radius: 6px;
    background: rgb(255 138 31 / 0.09);
    font-size: 11px;
    line-height: 1.4;
    color: var(--dim);
  }

  .body {
    display: flex;
    flex-direction: column;
  }
  section {
    padding: 15px 18px;
  }
  section + section {
    border-top: 1px solid var(--edge);
  }
  .head {
    display: flex;
    justify-content: space-between;
    align-items: baseline;
    gap: 10px;
    margin-bottom: 10px;
  }
  h2 {
    margin: 0;
    font-size: 10px;
    font-weight: 600;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    color: var(--faint);
  }
  .head span {
    font-family: var(--mono);
    font-size: 10px;
    color: var(--faint);
  }
  .slope {
    color: var(--signal) !important;
  }
  .slope.zero {
    color: var(--faint) !important;
  }

  .pills {
    display: flex;
    gap: 5px;
  }
  .pills button {
    flex: 1;
    padding: 8px 4px;
    border-radius: 7px;
    border: 1px solid var(--edge);
    background: transparent;
    color: var(--dim);
    font: inherit;
    font-size: 12px;
    cursor: pointer;
    transition: background 0.12s, color 0.12s;
  }
  .pills button:hover {
    background: rgb(255 255 255 / 0.05);
    color: var(--ink);
  }
  .pills button.on {
    background: var(--signal);
    border-color: var(--signal);
    color: #06131a;
    font-weight: 600;
  }

  .hrow {
    display: flex;
    align-items: center;
    gap: 12px;
  }
  .slider {
    flex: 1;
    min-width: 0;
    margin: 0;
    accent-color: var(--signal);
  }
  .num {
    width: 62px;
    padding: 6px 8px;
    border-radius: 7px;
    border: 1px solid var(--edge);
    background: var(--raise);
    color: var(--ink);
    font-family: var(--mono);
    font-size: 13px;
    font-variant-numeric: tabular-nums;
    text-align: right;
  }
  .num:focus-visible,
  .slider:focus-visible {
    outline: 2px solid var(--signal);
    outline-offset: 2px;
  }
  .ticks {
    display: flex;
    justify-content: space-between;
    margin-top: 7px;
    font-size: 10px;
    color: var(--faint);
  }

  .curve {
    width: 100%;
    height: auto;
    border-radius: 8px;
    border: 1px solid var(--edge);
    background: rgb(0 0 0 / 0.28);
  }
  .ground {
    stroke: var(--faint);
    stroke-width: 1;
    stroke-dasharray: 3 3;
  }
  .flight {
    fill: none;
    stroke: var(--signal);
    stroke-width: 1.5;
  }
  .from {
    fill: var(--own);
  }
  .to {
    fill: var(--signal);
  }
  .ref line {
    stroke: var(--faint);
    stroke-width: 1;
  }
  .ref text {
    fill: var(--faint);
    font-family: var(--mono);
    font-size: 8px;
  }

  .angles {
    display: flex;
    justify-content: space-between;
    margin-top: 8px;
    font-size: 11px;
    color: var(--faint);
  }
  .angles b {
    font-family: var(--mono);
    font-weight: 600;
    color: var(--ink);
  }

  .empty {
    margin: 0;
    padding: 26px 0;
    text-align: center;
    font-size: 11px;
    color: var(--faint);
  }

  /* --- Mobil: kartan fyller skärmen, panelen blir ett draglakan --------- */

  @media (max-width: 760px) {
    .viewer {
      display: block;
    }
    .stage {
      position: absolute;
      inset: 0;
    }
    .scalebar {
      left: 14px;
      bottom: auto;
      top: 18px;
    }
    .tools {
      top: 14px;
      right: 14px;
    }

    .panel {
      position: absolute;
      left: 0;
      right: 0;
      bottom: 0;
      max-height: 82dvh;
      border-left: 0;
      border-top: 1px solid var(--edge);
      border-radius: 16px 16px 0 0;
      background: rgb(10 26 32 / 0.93);
      backdrop-filter: blur(18px);
      box-shadow: 0 -14px 40px rgb(0 0 0 / 0.45);
      padding-bottom: env(safe-area-inset-bottom);
    }

    .grip {
      display: block;
      width: 100%;
      padding: 0;
      height: 26px;
      border: 0;
      background: transparent;
      cursor: pointer;
      position: relative;
      flex: none;
    }
    .grip::after {
      content: '';
      position: absolute;
      top: 9px;
      left: 50%;
      width: 38px;
      height: 4px;
      border-radius: 999px;
      background: rgb(255 255 255 / 0.24);
      transform: translateX(-50%);
    }

    .lede {
      display: none;
    }
    .dial {
      padding: 4px 16px 14px;
      border-bottom: 0;
    }
    .figure {
      margin: 0 0 10px;
      font-size: 36px;
    }
    .body {
      display: none;
      border-top: 1px solid var(--edge);
    }
    .panel.open .body {
      display: flex;
    }
    .panel.open .dial {
      border-bottom: 1px solid var(--edge);
    }

    .tools button {
      width: 42px;
      height: 42px;
      font-size: 17px;
    }
    .num {
      width: 68px;
      padding: 8px;
      font-size: 15px;
    }
    .slider {
      height: 28px;
    }
    .pills button {
      padding: 11px 4px;
      font-size: 13px;
    }
  }

  @media (prefers-reduced-motion: reduce) {
    .rule,
    .pills button {
      transition: none;
    }
  }
</style>
