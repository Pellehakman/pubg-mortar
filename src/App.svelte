<script>
  import MapViewer from './lib/MapViewer.svelte';
  import Login from './lib/Login.svelte';

  // Cosmetic gate only. These end up in the built JavaScript in plain text.
  const USER = 'phawkman';
  const PASS = 'mortar';
  const KEY = 'pmr.auth';

  let authed = $state(false);

  try {
    authed = localStorage.getItem(KEY) === '1';
  } catch {
    // storage blocked — just show the login
  }

  function check(user, pass) {
    if (user.trim().toLowerCase() !== USER || pass !== PASS) return false;
    authed = true;
    try {
      localStorage.setItem(KEY, '1');
    } catch {
      // not fatal, the session just won't be remembered
    }
    return true;
  }

  const maps = [
    { id: 'erangel', label: 'Erangel', src: '/erangel.webp', pixels: 4096, meters: 8000 },
    { id: 'miramar', label: 'Miramar', src: '/miramar.webp', pixels: 4096, meters: 8000 },
    { id: 'rondo', label: 'Rondo', src: '/rondo.webp', pixels: 4096, meters: 8120 },
    { id: 'taego', label: 'Taego', src: '/taego.webp', pixels: 4096, meters: 8000 }
  ];
</script>

{#if authed}
  <!-- Calibrated against: 403 m + 130 m → 486, and 503 m + 130 m → 612 -->
  <MapViewer {maps} spreadMeters={5} maxRange={700} arcShape={0.513} />
{:else}
  <Login onpass={check} />
{/if}