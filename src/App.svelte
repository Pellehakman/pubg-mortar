<script>
  import MapViewer from './lib/MapViewer.svelte';

  // pixels = image width in px, meters = real width the image covers.
  // Rondo is 8120 because its image includes a decorative border around
  // the playable area; the others run edge to edge.
  const maps = [
    { id: 'erangel', label: 'Erangel', src: '/erangel.webp', pixels: 4096, meters: 8000 },
    { id: 'miramar', label: 'Miramar', src: '/miramar.webp', pixels: 4096, meters: 8000 },
    { id: 'rondo', label: 'Rondo', src: '/rondo.webp', pixels: 4096, meters: 8120 },
    { id: 'taego', label: 'Taego', src: '/taego.webp', pixels: 4096, meters: 8000 }
  ];
</script>

<!-- Calibrated against: 403 m + 130 m → 486, and 503 m + 130 m → 612 -->
<MapViewer {maps} spreadMeters={5} maxRange={700} arcShape={0.513} />