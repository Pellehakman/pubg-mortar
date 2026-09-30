<script>
  import { untrack } from 'svelte';

  let {
    maps = [],
    spreadMeters = 5, // rough impact spread
    maxRange = 700, // mortar max range (highest entry in the range table)
    arcShape = 0.513 // how fast the arc flattens with range (1 = constant muzzle speed)
  } = $props();

  const G = 9.81;
  const MAX_SCALE = 6;
  const CLICK_SLOP = 5;
  const REMOVE_METERS = 100; // tap within this of the source to remove it

  // Range settings the mortar actually offers. Anything we compute snaps to
  // the nearest one, because those are the only numbers you can dial.
  const TABLE = [
    121, 133, 145, 157, 169, 181, 193, 204, 216, 228, 239, 250, 262, 273, 284, 295, 307, 317,
    328, 339, 350, 360, 371, 381, 391, 401, 411, 421, 431, 440, 450, 459, 468, 477, 487, 495,
    503, 512, 520, 528, 536, 544, 551, 559, 566, 573, 580, 587, 593, 600, 606, 612, 618, 624,
    629, 635, 639, 644, 649, 653, 658, 662, 666, 669, 673, 676, 679, 682, 685, 687, 689, 691,
    693, 695, 696, 697, 698, 699, 700
  ];
  const snap = (v) => TABLE.reduce((a, b) => (Math.abs(b - v) < Math.abs(a - v) ? b : a));

  // Elevation is a coarse call and the spread is around 10 m, so the slider
  // stops at seven fixed values rather than pretending to be exact.
  const H_STEPS = [-150, -75, -25, 0, 25, 75, 150];
  const LEVEL = 3;

  let container = $state(null);
  let containerW = $state(0);
  let containerH = $state(0);

  let mapId = $state(maps[0]?.id);
  const map = $derived(maps.find((m) => m.id === mapId) ?? maps[0]);
  const mpp = $derived(map ? map.meters / map.pixels : 1);

  let scale = $state(1);
  let tx = $state(0);
  let ty = $state(0);

  let origin = $state(null); // firing position
  let target = $state(null); // impact point
  let cursor = $state(null);
  let panning = $state(false);
  let sheetOpen = $state(false);

  let hIndex = $state(LEVEL);
  const deltaH = $derived(H_STEPS[hIndex]);

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
  const hitPx = $derived(Math.min(64, Math.max(18, (REMOVE_METERS / mpp) * scale)));

  // === Ballistics ========================================================
  //
  // The game's own range table follows S = 700·sin(2θ), running from 85° at
  // 121 m down to 45° at 700 m. Barrel angle is therefore read straight off
  // that relation. For height compensation we use a fitted arc:
  //
  //     θ = 90° − ½·arcsin( (S / maxRange)^arcShape )
  //
  // with the speed chosen so flat range always equals S exactly.
  // Calibrated against: 403 m + 130 m → 486, and 503 m + 130 m → 612.

  const ratioFor = (setting) => Math.pow(Math.min(1, Math.max(0, setting / maxRange)), arcShape);
  const angleFor = (setting) => Math.PI / 2 - 0.5 * Math.asin(ratioFor(setting));
  const speedFor = (setting) => (G * setting) / Math.max(1e-9, ratioFor(setting));

  function heightAt(x, setting) {
    const theta = angleFor(setting);
    const c = Math.cos(theta);
    return x * Math.tan(theta) - (G * x * x) / (2 * speedFor(setting) * c * c);
  }

  /**
   * Height at a fixed distance peaks partway up the range scale and then
   * falls, so find the peak first and bisect on the rising side.
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
    if (heightAt(distance, peak) < dh) return null;

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

    const ideal = dialFor(dist, deltaH);
    if (ideal === null) return { map: dist, bearing, dial: null, reason: 'far' };
    if (ideal < TABLE[0] - 6) return { map: dist, bearing, dial: null, reason: 'near' };
    return { map: dist, bearing, dial: snap(ideal), ideal };
  }

  const solution = $derived(solve(origin, target));
  const preview = $derived(origin && cursor && !panning ? solve(origin, cursor) : null);

  const overOrigin = $derived(
    !!(sOrigin && sCursor && Math.hypot(sCursor.x - sOrigin.x, sCursor.y - sOrigin.y) <= hitPx)
  );

  // === Trajectory profile, drawn 1:1 =====================================

  const profile = $derived.by(() => {
    if (!solution || solution.dial === null || solution.map < 1) return null;

    const D = solution.map;
    const S = solution.ideal;
    const W = 260;
    const H = 104;
    const pad = 9;

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
    const k = Math.min((W - pad * 2) / D, (H - pad * 2) / worldH); // one scale, both axes

    const ox = (W - D * k) / 2;
    const oy = pad + (H - pad * 2 - worldH * k) / 2;
    const px = (x) => ox + x * k;
    const py = (y) => oy + (maxY - y) * k;

    return {
      W,
      H,
      path: pts.map(([x, y]) => `${px(x).toFixed(1)},${py(y).toFixed(1)}`).join(' '),
      start: { x: px(0), y: py(0) },
      end: { x: px(D), y: py(deltaH) }
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
    clearAll();
    cursor = null;
    resetView();
  }

  function toImage(clientX, clientY) {
    const r = container.getBoundingClientRect();
    return { x: (clientX - r.left - tx) / scale, y: (clientY - r.top - ty) / scale };
  }

  // === Interaction =======================================================

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
      if (e.pointerType !== 'mouse') cursor = null; // no hover on touch
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

    if (sOrigin && Math.hypot(px - sOrigin.x, py - sOrigin.y) <= hitPx) {
      clearAll();
      return;
    }

    const p = { x: (px - tx) / scale, y: (py - ty) / scale };
    if (p.x < 0 || p.y < 0 || p.x > map.pixels || p.y > map.pixels) return;

    if (!origin) {
      origin = p;
      hIndex = LEVEL; // a new firing position starts from level ground
    } else {
      target = p;
    }
  }

  function clearAll() {
    origin = null;
    target = null;
    hIndex = LEVEL;
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

  const num = (m) => (m < 1000 ? `${Math.round(m)}` : `${(m / 1000).toFixed(2)}k`);
  const brg = (d) => `${String(Math.round(d) % 360).padStart(3, '0')}°`;
  const signed = (m) => `${m > 0 ? '+' : m < 0 ? '−' : ''}${Math.abs(m)}`;
  const why = (s) => (s?.reason === 'near' ? 'TOO CLOSE' : 'OUT OF RANGE');

  const SCALE_STEPS = [10, 25, 50, 100, 250, 500, 1000, 2000, 5000];
  const bar = $derived.by(() => {
    const mPerPx = mpp / scale;
    const meters = SCALE_STEPS.find((s) => s >= 120 * mPerPx) ?? SCALE_STEPS.at(-1);
    return { meters, width: meters / mPerPx };
  });
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
        <circle class="impact" cx={sTarget.x} cy={sTarget.y} r="2.5" />
      {/if}

      {#if sOrigin}
        {#if overOrigin}
          <circle class="hotzone" cx={sOrigin.x} cy={sOrigin.y} r={hitPx} />
        {/if}
        <circle class="origin-shadow" cx={sOrigin.x} cy={sOrigin.y} r="6.5" />
        <circle class="origin" class:armed={overOrigin} cx={sOrigin.x} cy={sOrigin.y} r="5" />
      {/if}
    </svg>

    {#if sOrigin && overOrigin}
      <div class="chip remove" style="left: {sOrigin.x}px; top: {sOrigin.y - hitPx - 10}px;">
        Click to remove
      </div>
    {/if}

    {#if sOrigin && sTarget && solution}
      <div
        class="chip"
        class:over={solution.dial === null}
        style="left: {(sOrigin.x + sTarget.x) / 2}px; top: {(sOrigin.y + sTarget.y) / 2}px;"
      >
        {#if solution.dial === null}
          {why(solution)}
        {:else}
          <strong>{solution.dial}</strong><em>{brg(solution.bearing)}</em>
        {/if}
      </div>
    {/if}

    {#if preview && sCursor && !overOrigin}
      <div class="ghost" style="left: {sCursor.x}px; top: {sCursor.y}px;">
        {#if preview.dial === null}
          {why(preview)}
        {:else}
          {num(preview.map)} m → <b>{preview.dial}</b>
        {/if}
      </div>
    {/if}

    <div class="tools">
      <button onclick={() => zoomCenter(1.4)} aria-label="Zoom in">+</button>
      <button onclick={() => zoomCenter(1 / 1.4)} aria-label="Zoom out">−</button>
      <button onclick={resetView} aria-label="Fit whole map">
        <svg viewBox="0 0 16 16" width="13" height="13" aria-hidden="true">
          <path
            d="M2 6V2h4M14 6V2h-4M2 10v4h4M14 10v4h-4"
            fill="none"
            stroke="currentColor"
            stroke-width="1.8"
          />
        </svg>
      </button>
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
      aria-label={sheetOpen ? 'Hide controls' : 'Show controls'}
    ></button>

    <header>
      <h1>Phawkman's Mortar&nbsp;Ruler</h1>
      {#if origin}<button class="clear" onclick={clearAll}>Clear</button>{/if}
    </header>

    <div class="display" class:live={!!solution?.dial} class:over={solution?.dial === null}>
      <span class="cap">Set range</span>
      {#if !solution}
        <p class="readout idle">- - -</p>
      {:else if solution.dial === null}
        <p class="readout alert">{why(solution)}</p>
      {:else}
        <p class="readout">{solution.dial}<small>m</small></p>
      {/if}
    </div>

    <div class="stats">
      <div><span class="cap">Distance</span><b>{solution ? `${num(solution.map)} m` : '—'}</b></div>
      <div><span class="cap">Bearing</span><b>{solution ? brg(solution.bearing) : '—'}</b></div>
      <div>
        <span class="cap">Elev</span><b class:on={deltaH !== 0}>{signed(deltaH)} m</b>
      </div>
    </div>

    {#if !origin}
      <p class="empty">Tap the map to set your firing position</p>
    {/if}

    <section class="elev">
      <span class="cap">Target elevation</span>
      <input
        class="slider"
        type="range"
        min="0"
        max={H_STEPS.length - 1}
        step="1"
        value={hIndex}
        oninput={(e) => (hIndex = Number(e.currentTarget.value))}
        aria-label="Target elevation relative to your position"
      />
      <div class="stops">
        {#each H_STEPS as v, i}
          <span class:on={i === hIndex}>{v === 0 ? '0' : Math.abs(v)}</span>
        {/each}
      </div>
      <div class="legend"><span>Below</span><span>Above</span></div>
    </section>

    <div class="body">
      {#if profile}
        <section class="arc">
          <span class="cap">Trajectory</span>
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
          </svg>
        </section>
      {/if}
    </div>

    <section class="maps">
      <span class="cap">Map</span>
      <div class="pills" role="group" aria-label="Choose map">
        {#each maps as m}
          <button class:on={m.id === mapId} onclick={() => chooseMap(m.id)}>{m.label}</button>
        {/each}
      </div>
    </section>
  </aside>
</div>

<style>
  .viewer {
    --void: #000305;
    --panel: #05090b;
    --hair: rgb(255 255 255 / 0.1);
    --ink: #e9f3f5;
    --dim: #8298a0;
    --faint: #5c7178;
    --neon: #2bff88;
    --warn: #ff4d4d;
    --sans: ui-sans-serif, system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif;
    --mono: ui-monospace, 'SF Mono', 'JetBrains Mono', Menlo, Consolas, monospace;

    position: relative;
    display: grid;
    grid-template-columns: 1fr 300px;
    height: 100%;
    background: var(--void);
    color: var(--ink);
    font-family: var(--sans);
    font-size: 14px;
    -webkit-font-smoothing: antialiased;
  }

  /* --- Map -------------------------------------------------------------- */

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
    fill: #fff;
  }
  .origin.armed {
    fill: var(--warn);
  }
  .origin-shadow {
    fill: rgb(0 0 0 / 0.75);
  }
  .hotzone {
    fill: rgb(255 77 77 / 0.12);
    stroke: var(--warn);
    stroke-width: 1.5;
    stroke-dasharray: 5 5;
  }
  .spread {
    fill: rgb(43 255 136 / 0.2);
    stroke: var(--neon);
    stroke-width: 2;
  }
  .spread-shadow {
    fill: none;
    stroke: #000;
    stroke-width: 4.5;
    opacity: 0.7;
  }
  .impact {
    fill: var(--neon);
  }
  .shot-shadow {
    stroke: #000;
    stroke-width: 5.5;
    opacity: 0.7;
  }
  .shot {
    stroke: var(--neon);
    stroke-width: 2;
    stroke-dasharray: 9 5;
  }
  .shot.over {
    stroke: var(--warn);
  }
  .aim {
    stroke: #fff;
    stroke-width: 1.2;
    stroke-dasharray: 2 6;
    opacity: 0.4;
  }

  .chip {
    position: absolute;
    transform: translate(-50%, -50%);
    display: flex;
    align-items: baseline;
    gap: 8px;
    padding: 5px 11px;
    border-radius: 4px;
    background: rgb(0 3 5 / 0.92);
    border: 1px solid var(--neon);
    font-family: var(--mono);
    font-size: 15px;
    font-weight: 700;
    font-variant-numeric: tabular-nums;
    color: var(--neon);
    white-space: nowrap;
    pointer-events: none;
  }
  .chip em {
    font-style: normal;
    font-size: 11px;
    font-weight: 400;
    color: var(--dim);
  }
  .chip.over,
  .chip.remove {
    border-color: var(--warn);
    color: var(--warn);
    font-family: var(--sans);
    font-size: 12px;
    font-weight: 600;
    letter-spacing: 0.06em;
  }
  .chip.remove {
    transform: translate(-50%, -100%);
  }

  .ghost {
    position: absolute;
    transform: translate(16px, 16px);
    font-family: var(--mono);
    font-size: 12px;
    font-weight: 600;
    font-variant-numeric: tabular-nums;
    color: rgb(255 255 255 / 0.75);
    text-shadow: 0 0 5px #000, 0 0 2px #000;
    white-space: nowrap;
    pointer-events: none;
  }
  .ghost b {
    color: var(--neon);
  }

  .tools {
    position: absolute;
    top: 14px;
    right: 14px;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    border-radius: 8px;
    border: 1px solid var(--hair);
    background: rgb(0 3 5 / 0.82);
    backdrop-filter: blur(10px);
  }
  .tools button {
    width: 36px;
    height: 36px;
    display: grid;
    place-items: center;
    padding: 0;
    border: 0;
    border-radius: 0;
    background: transparent;
    color: var(--ink);
    font-size: 16px;
    line-height: 1;
    cursor: pointer;
  }
  .tools button + button {
    box-shadow: inset 0 1px 0 var(--hair);
  }
  .tools button:hover {
    background: rgb(255 255 255 / 0.1);
    color: var(--neon);
  }

  .scalebar {
    position: absolute;
    left: 16px;
    bottom: 16px;
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: var(--mono);
    font-size: 12px;
    font-weight: 600;
    text-shadow: 0 0 6px #000, 0 0 2px #000;
    pointer-events: none;
  }
  .rule {
    height: 7px;
    border: 1.5px solid #fff;
    border-top: 0;
    transition: width 0.1s linear;
  }

  /* --- Panel: flat rows divided by hairlines, no nested boxes ----------- */

  .panel {
    display: flex;
    flex-direction: column;
    min-height: 0;
    background: var(--panel);
    border-left: 1px solid var(--hair);
    overflow-y: auto;
    overscroll-behavior: contain;
  }
  .grip {
    display: none;
  }

  .cap {
    display: block;
    font-size: 10px;
    font-weight: 600;
    letter-spacing: 0.18em;
    text-transform: uppercase;
    color: var(--faint);
  }

  header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    padding: 14px 18px;
  }
  h1 {
    margin: 0;
    font-size: 10px;
    font-weight: 700;
    letter-spacing: 0.18em;
    text-transform: uppercase;
    color: var(--faint);
  }
  .clear {
    flex: none;
    padding: 5px 11px;
    border-radius: 4px;
    border: 1px solid var(--warn);
    background: transparent;
    color: var(--warn);
    font: inherit;
    font-size: 11px;
    font-weight: 600;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    cursor: pointer;
  }
  .clear:hover {
    background: var(--warn);
    color: #000;
  }

  /* The main readout — a lit display on black */
  .display {
    padding: 4px 18px 22px;
    background: var(--void);
    border-bottom: 1px solid var(--hair);
  }
  .readout {
    margin: 6px 0 0;
    font-family: var(--mono);
    font-size: 76px;
    font-weight: 700;
    line-height: 0.9;
    letter-spacing: -0.04em;
    font-variant-numeric: tabular-nums;
    color: var(--neon);
    text-shadow: 0 0 28px rgb(43 255 136 / 0.45);
  }
  .readout small {
    margin-left: 6px;
    font-size: 20px;
    font-weight: 600;
    letter-spacing: 0;
    color: var(--faint);
    text-shadow: none;
  }
  .readout.idle {
    color: rgb(43 255 136 / 0.18);
    text-shadow: none;
  }
  .readout.alert {
    font-family: var(--sans);
    font-size: 24px;
    letter-spacing: 0.04em;
    color: var(--warn);
    text-shadow: 0 0 22px rgb(255 77 77 / 0.4);
  }

  .stats {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
  }
  .stats div {
    padding: 11px 18px;
  }
  .stats div + div {
    border-left: 1px solid var(--hair);
    padding-left: 14px;
  }
  .stats b {
    display: block;
    margin-top: 5px;
    font-family: var(--mono);
    font-size: 17px;
    font-weight: 700;
    font-variant-numeric: tabular-nums;
    color: var(--ink);
  }
  .stats b.on {
    color: var(--neon);
  }

  .empty {
    margin: 0;
    padding: 16px 18px;
    border-top: 1px solid var(--hair);
    font-size: 13px;
    color: var(--faint);
  }

  .body {
    display: flex;
    flex-direction: column;
  }
  section {
    padding: 16px 18px;
    border-top: 1px solid var(--hair);
  }
  .maps {
    margin-top: auto;
  }

  .slider {
    width: 100%;
    margin: 12px 0 0;
    accent-color: var(--neon);
  }
  .slider:focus-visible {
    outline: 2px solid var(--neon);
    outline-offset: 3px;
  }
  .stops {
    display: flex;
    justify-content: space-between;
    margin-top: 6px;
    font-family: var(--mono);
    font-size: 11px;
    font-variant-numeric: tabular-nums;
    color: var(--faint);
  }
  .stops span {
    flex: 1;
    text-align: center;
  }
  .stops span.on {
    color: var(--neon);
    font-weight: 700;
  }
  .legend {
    display: flex;
    justify-content: space-between;
    margin-top: 4px;
    font-size: 10px;
    letter-spacing: 0.1em;
    text-transform: uppercase;
    color: var(--faint);
  }

  .curve {
    width: 100%;
    height: auto;
    margin-top: 8px;
  }
  .ground {
    stroke: var(--faint);
    stroke-width: 1.2;
    stroke-dasharray: 3 3;
  }
  .flight {
    fill: none;
    stroke: var(--neon);
    stroke-width: 2;
  }
  .from {
    fill: #fff;
  }
  .to {
    fill: var(--neon);
  }

  .pills {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-top: 10px;
  }
  .pills button {
    flex: 1 1 calc(50% - 3px);
    padding: 10px 6px;
    border-radius: 5px;
    border: 1px solid var(--hair);
    background: transparent;
    color: var(--dim);
    font: inherit;
    font-size: 13px;
    font-weight: 600;
    cursor: pointer;
  }
  .pills button:hover {
    border-color: var(--dim);
    color: var(--ink);
  }
  .pills button.on {
    background: var(--neon);
    border-color: var(--neon);
    color: #000;
  }

  /* --- Mobile ----------------------------------------------------------- */

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

    .panel {
      position: absolute;
      left: 0;
      right: 0;
      bottom: 0;
      max-height: 84dvh;
      border-left: 0;
      border-top: 1px solid var(--hair);
      border-radius: 14px 14px 0 0;
      background: rgb(5 9 11 / 0.97);
      backdrop-filter: blur(18px);
      box-shadow: 0 -14px 44px rgb(0 0 0 / 0.6);
      padding-bottom: env(safe-area-inset-bottom);
    }
    .grip {
      display: block;
      position: relative;
      flex: none;
      width: 100%;
      height: 24px;
      padding: 0;
      border: 0;
      background: transparent;
      cursor: pointer;
    }
    .grip::after {
      content: '';
      position: absolute;
      top: 9px;
      left: 50%;
      width: 40px;
      height: 4px;
      border-radius: 999px;
      background: rgb(255 255 255 / 0.28);
      transform: translateX(-50%);
    }
    header {
      padding: 2px 18px 10px;
    }
    .display {
      padding: 4px 18px 18px;
      background: transparent;
    }
    .readout {
      font-size: 46px;
    }
    .stats {
      display: none;
    }
    .body,
    .maps {
      display: none;
    }
    .panel.open .body {
      display: flex;
    }
    .panel.open .maps {
      display: block;
    }

    .tools button {
      width: 44px;
      height: 44px;
      font-size: 18px;
    }
    .slider {
      height: 30px;
    }
    .pills button {
      padding: 13px 6px;
      font-size: 14px;
    }
  }

  @media (prefers-reduced-motion: reduce) {
    .rule {
      transition: none;
    }
  }
</style>