<script lang="ts">
	import { onMount } from 'svelte';
	import 'leaflet/dist/leaflet.css';

	let L;
	let map;

	onMount(async () => {
		L = await import('leaflet');

		map = L.map('map').setView([51.1657, 10.4515], 6);

		L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
			maxZoom: 19,
			attribution: '© OpenStreetMap contributors'
		}).addTo(map);
	});
</script>

<div id="map" class="fixed inset-0"></div>

<style>
	.leaflet-tile {
		image-rendering: pixelated;
		will-change: transform;
		box-shadow: none !important;
		margin: 0 !important;
		padding: 0 !important;
		border: 0 !important;
		display: block;
		backface-visibility: hidden;
		-webkit-backface-visibility: hidden;
	}
</style>
