<script>
  import MapViewer from './lib/MapViewer.svelte';

  // Cosmetic gate only — these end up in the built JavaScript in plain text.
  const USER = 'admin';
  const PASS = 'change-me';

  let authed = $state(false);
  let user = $state('');
  let pass = $state('');
  let failed = $state(false);

  // Remember the login for this tab. Wrapped because storage can be blocked.
  try {
    authed = sessionStorage.getItem('pmr') === '1';
  } catch {
    authed = false;
  }

  function submit(e) {
    e.preventDefault();
    if (user.trim().toLowerCase() === USER && pass === PASS) {
      authed = true;
      try {
        sessionStorage.setItem('pmr', '1');
      } catch {
        // fine, it just won't be remembered
      }
      return;
    }
    failed = true;
    pass = '';
  }

  // pixels = image width in px, meters = real width the image covers.
  // Rondo is 8120 because its image includes a border around the play area.
  const maps = [
    { id: 'erangel', label: 'Erangel', src: '/erangel.webp', pixels: 4096, meters: 8000 },
    { id: 'miramar', label: 'Miramar', src: '/miramar.webp', pixels: 4096, meters: 8000 },
    { id: 'rondo', label: 'Rondo', src: '/rondo.webp', pixels: 4096, meters: 8120 },
    { id: 'taego', label: 'Taego', src: '/taego.webp', pixels: 4096, meters: 8000 }
  ];
</script>

{#if !authed}
  <div class="gate">
    <form onsubmit={submit}>
      <h1>Phawkman's Mortar&nbsp;Ruler</h1>
      <input placeholder="Username" autocomplete="username" bind:value={user} />
      <input
        type="password"
        placeholder="Password"
        autocomplete="current-password"
        bind:value={pass}
      />
      <button type="submit">Enter</button>
      {#if failed}<p>Wrong username or password</p>{/if}
    </form>
  </div>
{:else}
  <!-- Calibrated against: 403 m + 130 m → 486, and 503 m + 130 m → 612 -->
  <MapViewer {maps} spreadMeters={5} maxRange={700} arcShape={0.513} />
{/if}

<style>
  .gate {
    display: grid;
    place-items: center;
    height: 100%;
    padding: 24px;
    background: #000305;
    font-family: ui-sans-serif, system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif;
  }
  form {
    display: flex;
    flex-direction: column;
    gap: 12px;
    width: 100%;
    max-width: 290px;
  }
  h1 {
    margin: 0 0 8px;
    font-size: 11px;
    font-weight: 700;
    letter-spacing: 0.18em;
    text-transform: uppercase;
    color: #2bff88;
    text-shadow: 0 0 22px rgb(43 255 136 / 0.4);
  }
  input {
    padding: 12px;
    border: 1px solid rgb(255 255 255 / 0.14);
    border-radius: 5px;
    background: #05090b;
    color: #e9f3f5;
    font-family: ui-monospace, 'SF Mono', Menlo, Consolas, monospace;
    font-size: 16px; /* 16px stops iOS from zooming the page on focus */
  }
  input:focus {
    outline: none;
    border-color: #2bff88;
  }
  button {
    margin-top: 4px;
    padding: 12px;
    border: 1px solid #2bff88;
    border-radius: 5px;
    background: #2bff88;
    color: #000;
    font: inherit;
    font-size: 13px;
    font-weight: 700;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    cursor: pointer;
  }
  button:hover {
    background: transparent;
    color: #2bff88;
  }
  p {
    margin: 0;
    font-size: 12px;
    color: #ff4d4d;
  }
</style>