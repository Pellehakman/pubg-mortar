<script>
  import MapViewer from './lib/MapViewer.svelte';

  // pixels = bildens bredd i px, meters = kartans verkliga bredd i meter.
  // Skalan räknas ut därifrån, så bilderna behöver inte vara lika stora.
 const maps = [
  { id: 'erangel', label: 'Erangel', src: '/erangel.webp', pixels: 4096, meters: 8000 },
  { id: 'miramar', label: 'Miramar', src: '/miramar.webp', pixels: 4096, meters: 8000 }
];
</script>

<!-- Kalibrerat mot: 403 m + 130 m → 486, och 503 m + 130 m → 612 -->
<MapViewer {maps} spreadMeters={5} maxRange={696} arcShape={0.53} />
