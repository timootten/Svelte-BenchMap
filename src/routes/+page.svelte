<script lang="ts">
	import { onMount } from 'svelte';

	let L;
	let map;
	let benchesLayer;

	async function fetchBenches(bbox: string) {
		const query = `
      [out:json][timeout:25];
      node["amenity"="bench"](${bbox});
      out body;
    `;

		const url = 'https://overpass-api.de/api/interpreter?data=' + encodeURIComponent(query);
		const res = await fetch(url);
		const data = await res.json();
		return data.elements;
	}

	function updateBenches(bbox) {
		fetchBenches(bbox).then((benches) => {
			benchesLayer.clearLayers();
			benches.forEach((bench) => {
				if (bench.lat && bench.lon) {
					L.marker([bench.lat, bench.lon]).addTo(benchesLayer);
				}
			});
		});
	}

	onMount(async () => {
		L = await import('leaflet');

		map = L.map('map').setView([51.1657, 10.4515], 6); // Deutschlandzentriert

		L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
			maxZoom: 19,
			attribution: '© OpenStreetMap contributors'
		}).addTo(map);

		benchesLayer = L.layerGroup().addTo(map);

		updateBenches(map.getBounds().toBBoxString());

		map.on('moveend', () => {
			updateBenches(map.getBounds().toBBoxString());
		});
	});
</script>

<div id="map"></div>

<style>
	#map {
		height: 100vh;
		width: 100vw;
	}
</style>
