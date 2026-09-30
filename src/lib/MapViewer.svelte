<script>
  import { untrack } from 'svelte';

  let {
    maps = [],
    spreadMeters = 5, // rough impact spread
    maxRange = 696, // mortar max range
    arcShape = 0.53 // how fast the arc flattens with range (1 = constant muzzle speed)
  } = $props();

  const G = 9.81;
  const MAX_SCALE = 6;
  const CLICK_SLOP = 5;
  const REMOVE_METERS = 100; // tap within this of the source to remove it
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

  let origin = $state(null); // firing position
  let target = $state(null); // impact point
  let cursor = $state(null);
  let panning = $state(false);
  let sheetOpen = $state(false);

  let deltaH = $state(0); // negative = target sits lower

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

  // Removal hotspot: REMOVE_METERS on the map, kept tappable when zoomed
  // out and kept sane when zoomed in.
  const hitPx = $derived(Math.min(64, Math.max(18, (REMOVE_METERS / mpp) * scale)));

  // === Ballistics ========================================================
  //
  // The mortar takes the high solution. Barrel angle for range setting S:
  //
  //     θ = 90° − ½·arcsin( (S / maxRange)^arcShape )
  //
  // and the speed is chosen so flat range equals S exactly, which keeps
  // level shots correct no matter how the parameters are tuned.
  // arcShape = 1 means constant muzzle speed.
  //
  // Calibrated against: 403 m + 130 m → 486, and 503 m + 130 m → 612.

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
   * Height at a fixed distance does not grow all the way to max range — it
   * peaks and then falls. Golden-section search finds the peak, then we
   * bisect on the rising side. Without this, steep uphill shots are all
   * reported as out of range.
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
    const dial = dialFor(dist, deltaH);
    if (dial === null) return { map: dist, bearing, dial: null, launch: null, impact: null };
    return {
      map: dist,
      bearing,
      dial,
      launch: (angleFor(dial) * 180) / Math.PI,
      impact: impactAngle(dist, dial)
    };
  }

  const solution = $derived(solve(origin, target));
  const preview = $derived(origin && cursor && !panning ? solve(origin, cursor) : null);

  const overOrigin = $derived(
    !!(sOrigin && sCursor && Math.hypot(sCursor.x - sOrigin.x, sCursor.y - sOrigin.y) <= hitPx)
  );

  // === Trajectory profile, drawn 1:1 =====================================

  const BAR_STEPS = [25, 50, 100, 200, 400];

  const profile = $derived.by(() => {
    if (!solution || solution.dial === null || solution.map < 1) return null;

    const D = solution.map;
    const S = solution.dial;
    const W = 252;
    const H = 132;
    const pad = 10;
    const foot = 16;

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
    const k = Math.min(boxW / D, boxH / worldH); // one scale for both axes

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
      bar: { meters: barMeters, width: barMeters * k, x: pad, y: H - 7 }
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
      deltaH = 0; // a new firing position starts from level ground
    } else {
      target = p;
    }
  }

  function clearAll() {
    origin = null;
    target = null;
    deltaH = 0;
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

  const fmt = (m) => (m < 1000 ? `${Math.round(m)}` : `${(m / 1000).toFixed(2)}k`);
  const brg = (d) => `${String(Math.round(d) % 360).padStart(3, '0')}°`;
  const signed = (m) => `${m >= 0 ? '+' : '−'}${Math.round(Math.abs(m))} m`;

  const STEPS = [10, 25, 50, 100, 250, 500, 1000, 2000, 5000];
  const bar = $derived.by(() => {
    const mPerPx = mpp / scale;
    const meters = STEPS.find((s) => s >= 120 * mPerPx) ?? STEPS[STEPS.length - 1];
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
          out of range
        {:else}
          <strong>{fmt(solution.dial)} m</strong><em>{brg(solution.bearing)}</em>
        {/if}
      </div>
    {/if}

    {#if preview && sCursor && !overOrigin}
      <div class="ghost" style="left: {sCursor.x}px; top: {sCursor.y}px;">
        {#if preview.dial === null}
          out of range
        {:else if deltaH !== 0}
          {fmt(preview.map)} → <b>{fmt(preview.dial)} m</b>
        {:else}
          {fmt(preview.map)} m
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

    <div class="bar">
      <h1>Phawkman's Mortar&nbsp;Ruler</h1>
      {#if origin}
        <button class="clear" onclick={clearAll}>Clear</button>
      {/if}
    </div>

    <div class="numbers">
      <div class="big">
        <span class="cap">Distance</span>
        <p class="fig">
          {#if solution}{fmt(solution.map)}<small>m</small>{:else}<span class="void">—</span>{/if}
        </p>
      </div>
      <div class="big set" class:over={solution?.dial === null}>
        <span class="cap">Set range</span>
        <p class="fig">
          {#if !solution}
            <span class="void">—</span>
          {:else if solution.dial === null}
            <span class="oor">out of range</span>
          {:else}
            {fmt(solution.dial)}<small>m</small>
          {/if}
        </p>
      </div>
    </div>

    <div class="pair">
      <div><span class="cap">Bearing</span><b>{solution ? brg(solution.bearing) : '—'}</b></div>
      <div>
        <span class="cap">Elevation</span>
        <b class:accent={deltaH !== 0}>{deltaH === 0 ? '0 m' : signed(deltaH)}</b>
      </div>
    </div>

    {#if !origin}
      <p class="empty">Tap the map to set your firing position</p>
    {/if}

    <div class="body">
      <section>
        <span class="cap">Target elevation</span>
        <div class="hrow">
          <input
            class="slider"
            type="range"
            min={-H_LIMIT}
            max={H_LIMIT}
            step="10"
            value={deltaH}
            oninput={(e) => setDelta(e.currentTarget.value)}
            aria-label="Elevation difference in metres, negative when the target is lower"
          />
          <input
            class="num"
            type="number"
            min={-H_LIMIT}
            max={H_LIMIT}
            step="1"
            value={deltaH}
            onchange={(e) => setDelta(e.currentTarget.value)}
            aria-label="Elevation difference, exact value"
          />
        </div>
        <div class="ticks"><span>−{H_LIMIT} below</span><span>above +{H_LIMIT}</span></div>
      </section>

      {#if profile && solution}
        <section>
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
          <p class="caption">
            Apex {profile.apex} m · Impact {Math.round(solution.impact)}°
          </p>
        </section>
      {/if}

      <section class="maps">
        <span class="cap">Map</span>
        <div class="pills" role="group" aria-label="Choose map">
          {#each maps as m}
            <button class:on={m.id === mapId} onclick={() => chooseMap(m.id)}>{m.label}</button>
          {/each}
        </div>
      </section>
    </div>
  </aside>
</div>

<style>
  .viewer {
    --bg: #030c10;
    --panel: #07171d;
    --raise: #0f2b34;
    --edge: rgb(255 255 255 / 0.16);
    --ink: #ffffff;
    --dim: #b3ccd4;
    --faint: #7c9aa5;
    --signal: #ff9522;
    --own: #ffffff;
    --warn: #ff5242;
    --sans: ui-sans-serif, system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif;
    --mono: ui-monospace, 'SF Mono', 'JetBrains Mono', Menlo, Consolas, monospace;

    position: relative;
    display: grid;
    grid-template-columns: 1fr 300px;
    height: 100%;
    background: var(--bg);
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
    fill: var(--own);
  }
  .origin.armed {
    fill: var(--warn);
  }
  .origin-shadow {
    fill: rgb(0 0 0 / 0.7);
  }
  .hotzone {
    fill: rgb(255 82 66 / 0.14);
    stroke: var(--warn);
    stroke-width: 1.5;
    stroke-dasharray: 5 5;
  }

  .spread {
    fill: rgb(255 149 34 / 0.2);
    stroke: var(--signal);
    stroke-width: 2;
  }
  .spread-shadow {
    fill: none;
    stroke: #000;
    stroke-width: 4;
    opacity: 0.6;
  }
  .impact {
    fill: var(--signal);
  }
  .shot-shadow {
    stroke: #000;
    stroke-width: 5;
    opacity: 0.6;
  }
  .shot {
    stroke: var(--signal);
    stroke-width: 2;
    stroke-dasharray: 9 5;
  }
  .shot.over {
    stroke: var(--warn);
  }
  .aim {
    stroke: var(--own);
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
    border-radius: 999px;
    background: rgb(3 12 16 / 0.92);
    border: 1.5px solid var(--signal);
    font-family: var(--mono);
    font-size: 14px;
    font-weight: 600;
    font-variant-numeric: tabular-nums;
    white-space: nowrap;
    pointer-events: none;
  }
  .chip em {
    font-style: normal;
    font-size: 11px;
    font-weight: 400;
    color: var(--dim);
  }
  .chip.over {
    border-color: var(--warn);
    color: var(--warn);
  }
  .chip.remove {
    transform: translate(-50%, -100%);
    border-color: var(--warn);
    color: var(--warn);
    font-family: var(--sans);
    font-size: 12px;
  }

  .ghost {
    position: absolute;
    transform: translate(16px, 16px);
    font-family: var(--mono);
    font-size: 12px;
    font-weight: 600;
    font-variant-numeric: tabular-nums;
    color: rgb(255 255 255 / 0.8);
    text-shadow: 0 0 5px #000, 0 0 2px #000;
    white-space: nowrap;
    pointer-events: none;
  }
  .ghost b {
    color: var(--signal);
  }

  .tools {
    position: absolute;
    top: 14px;
    right: 14px;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    border-radius: 10px;
    border: 1px solid var(--edge);
    background: rgb(3 12 16 / 0.82);
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
    box-shadow: inset 0 1px 0 var(--edge);
  }
  .tools button:hover {
    background: rgb(255 255 255 / 0.12);
    color: var(--signal);
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
    border: 1.5px solid var(--ink);
    border-top: 0;
    transition: width 0.1s linear;
  }

  /* --- Panel ------------------------------------------------------------ */

  .panel {
    display: flex;
    flex-direction: column;
    min-height: 0;
    background: var(--panel);
    border-left: 1px solid var(--edge);
    overflow-y: auto;
    overscroll-behavior: contain;
  }
  .grip {
    display: none;
  }

  .bar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    padding: 14px 16px;
    border-bottom: 1px solid var(--edge);
  }
  h1 {
    margin: 0;
    font-size: 11px;
    font-weight: 700;
    letter-spacing: 0.1em;
    text-transform: uppercase;
    color: var(--dim);
  }
  .clear {
    flex: none;
    padding: 6px 12px;
    border-radius: 999px;
    border: 1.5px solid var(--warn);
    background: transparent;
    color: var(--warn);
    font: inherit;
    font-size: 12px;
    font-weight: 600;
    cursor: pointer;
  }
  .clear:hover {
    background: var(--warn);
    color: #030c10;
  }

  .numbers {
    display: flex;
    flex-direction: column;
  }
  .big {
    padding: 14px 16px;
  }
  .big + .big {
    border-top: 1px solid var(--edge);
  }
  .cap {
    display: block;
    font-size: 11px;
    font-weight: 600;
    letter-spacing: 0.1em;
    text-transform: uppercase;
    color: var(--faint);
  }
  .fig {
    margin: 4px 0 0;
    font-family: var(--mono);
    font-size: 46px;
    font-weight: 700;
    line-height: 1;
    letter-spacing: -0.03em;
    font-variant-numeric: tabular-nums;
    color: var(--ink);
  }
  .set .fig {
    color: var(--signal);
  }
  .fig small {
    margin-left: 4px;
    font-size: 17px;
    font-weight: 600;
    color: var(--faint);
  }
  .void {
    color: var(--faint);
  }
  .oor {
    font-size: 20px;
    letter-spacing: 0;
    color: var(--warn);
  }

  .pair {
    display: grid;
    grid-template-columns: 1fr 1fr;
    border-top: 1px solid var(--edge);
    border-bottom: 1px solid var(--edge);
  }
  .pair div {
    padding: 11px 16px;
  }
  .pair div + div {
    border-left: 1px solid var(--edge);
  }
  .pair b {
    display: block;
    margin-top: 4px;
    font-family: var(--mono);
    font-size: 20px;
    font-weight: 700;
    font-variant-numeric: tabular-nums;
  }
  .pair b.accent {
    color: var(--signal);
  }

  .empty {
    margin: 0;
    padding: 16px;
    font-size: 13px;
    line-height: 1.5;
    color: var(--faint);
  }

  .body {
    display: flex;
    flex-direction: column;
  }
  section {
    padding: 14px 16px;
  }
  section + section {
    border-top: 1px solid var(--edge);
  }
  .maps {
    margin-top: auto;
  }

  .hrow {
    display: flex;
    align-items: center;
    gap: 12px;
    margin-top: 10px;
  }
  .slider {
    flex: 1;
    min-width: 0;
    margin: 0;
    accent-color: var(--signal);
  }
  .num {
    width: 68px;
    padding: 7px 9px;
    border-radius: 8px;
    border: 1.5px solid var(--edge);
    background: var(--raise);
    color: var(--ink);
    font-family: var(--mono);
    font-size: 15px;
    font-weight: 600;
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
    margin-top: 8px;
    font-size: 11px;
    color: var(--faint);
  }

  .curve {
    width: 100%;
    height: auto;
    margin-top: 10px;
    border-radius: 8px;
    border: 1px solid var(--edge);
    background: rgb(0 0 0 / 0.4);
  }
  .ground {
    stroke: var(--faint);
    stroke-width: 1.2;
    stroke-dasharray: 3 3;
  }
  .flight {
    fill: none;
    stroke: var(--signal);
    stroke-width: 2;
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
  .caption {
    margin: 8px 0 0;
    font-family: var(--mono);
    font-size: 12px;
    color: var(--dim);
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
    border-radius: 8px;
    border: 1.5px solid var(--edge);
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
    background: var(--signal);
    border-color: var(--signal);
    color: #030c10;
  }

  /* --- Mobile: map fills the screen, panel becomes a sheet --------------- */

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
      border-top: 1px solid var(--edge);
      border-radius: 16px 16px 0 0;
      background: rgb(7 23 29 / 0.96);
      backdrop-filter: blur(18px);
      box-shadow: 0 -14px 44px rgb(0 0 0 / 0.55);
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
      background: rgb(255 255 255 / 0.3);
      transform: translateX(-50%);
    }

    .bar {
      padding: 2px 16px 12px;
      border-bottom: 0;
    }
    .numbers {
      flex-direction: row;
      border-top: 1px solid var(--edge);
    }
    .big {
      flex: 1;
      padding: 12px 16px;
    }
    .big + .big {
      border-top: 0;
      border-left: 1px solid var(--edge);
    }
    .fig {
      font-size: 38px;
    }
    .body {
      display: none;
    }
    .panel.open .body {
      display: flex;
    }

    .tools button {
      width: 44px;
      height: 44px;
      font-size: 18px;
    }
    .num {
      width: 74px;
      padding: 9px;
      font-size: 16px;
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