<script lang="ts">
import { onMount, onDestroy } from 'svelte';
import 'leaflet/dist/leaflet.css';
import type { Map as LeafletMap, LayerGroup } from 'leaflet';

let L: any;
let map: LeafletMap;
let benchesLayer: LayerGroup | null = null;

const TILE_SIZE = 0.05; // ~5km
let loadedAreas = new Set<string>();
let benchCache = new Map<string, any[]>(); // Cache für geladene Bänke

let loading = false;
let zoomLevel = 0;
let loadedBenchesCount = 0;

// Debounce Timer
let debounceTimer: number;

// Hilfsfunktionen
function roundCoord(n: number): number {
    return Math.floor(n / TILE_SIZE) * TILE_SIZE;
}

function getTileKey(lat: number, lon: number): string {
    return `${roundCoord(lat)}_${roundCoord(lon)}`;
}

function getAreaKeys(bounds: any): string[] {
    const keys: string[] = [];
    for (let lat = roundCoord(bounds.getSouth()); lat <= roundCoord(bounds.getNorth()); lat += TILE_SIZE) {
        for (let lon = roundCoord(bounds.getWest()); lon <= roundCoord(bounds.getEast()); lon += TILE_SIZE) {
            keys.push(getTileKey(lat, lon));
        }
    }
    return keys;
}

function clearAllData() {
    loadedAreas.clear();
    benchCache.clear();
    loadedBenchesCount = 0;
    if (benchesLayer) {
        benchesLayer.clearLayers();
    }
}

function debounceMapUpdate() {
    clearTimeout(debounceTimer);
    debounceTimer = setTimeout(() => {
        zoomLevel = map.getZoom();
        
        if (zoomLevel >= 14) {
            loadBenches();
        } else {
            // WICHTIG: Hier werden auch die geladenen Bereiche geleert
            clearAllData();
        }
    }, 150); // 150ms debounce
}

onMount(async () => {
    try {
        L = await import('leaflet');
        
        map = L.map('map').setView([51.1657, 10.4515], 6);
        
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            maxZoom: 19,
            attribution: '© OpenStreetMap contributors'
        }).addTo(map);
        
        benchesLayer = L.layerGroup().addTo(map);
        
        // Event-Handler mit Debouncing
        map.on('moveend zoomend', debounceMapUpdate);
        
    } catch (error) {
        console.error('Fehler beim Initialisieren der Karte:', error);
    }
});

onDestroy(() => {
    clearTimeout(debounceTimer);
    if (map) {
        map.remove();
    }
});

async function loadBenches() {
    if (!map || !benchesLayer) return;
    
    const bounds = map.getBounds();
    const areaKeys = getAreaKeys(bounds);
    
    const toLoad = areaKeys.filter((key) => !loadedAreas.has(key));
    if (toLoad.length === 0) return;
    
    loading = true;
    
    try {
        // Parallelisierte Anfragen für bessere Performance
        const promises = toLoad.map(async (key) => {
            // Prüfen ob bereits im Cache
            if (benchCache.has(key)) {
                const cachedBenches = benchCache.get(key)!;
                addBenchesToMap(cachedBenches);
                loadedAreas.add(key);
                return;
            }
            
            const [latStr, lonStr] = key.split('_');
            const lat = parseFloat(latStr);
            const lon = parseFloat(lonStr);
            const bbox = `${lat},${lon},${lat + TILE_SIZE},${lon + TILE_SIZE}`;
            
            const query = `
                [out:json][timeout:15];
                node["amenity"="bench"](${bbox});
                out body;
            `;
            
            try {
                const res = await fetch('https://overpass-api.de/api/interpreter', {
                    method: 'POST',
                    body: query,
                    headers: {
                        'Content-Type': 'text/plain'
                    }
                });
                
                if (!res.ok) {
                    throw new Error(`HTTP ${res.status}`);
                }
                
                const data = await res.json();
                
                // Cache die Ergebnisse
                benchCache.set(key, data.elements);
                addBenchesToMap(data.elements);
                loadedAreas.add(key);
                
            } catch (err) {
                console.warn(`Fehler bei Tile ${key}:`, err);
            }
        });
        
        await Promise.all(promises);
        
    } catch (error) {
        console.error('Fehler beim Laden der Bänke:', error);
    } finally {
        loading = false;
    }
}

function addBenchesToMap(benches: any[]) {
    if (!benchesLayer || !L) return;
    
    benches.forEach(bench => {
        const marker = L.circleMarker([bench.lat, bench.lon], {
            radius: 4,
            color: '#008000',
            weight: 1,
            fillOpacity: 0.7,
            fillColor: '#00ff00'
        });
        
        // Optional: Popup mit Informationen
        if (bench.tags && Object.keys(bench.tags).length > 0) {
            const popupContent = Object.entries(bench.tags)
                .map(([key, value]) => `<strong>${key}:</strong> ${value}`)
                .join('<br>');
            marker.bindPopup(popupContent);
        }
        
        marker.addTo(benchesLayer);
        loadedBenchesCount++;
    });
}
</script>

<!-- Map Container -->
<div id="map" class="fixed inset-0 z-0"></div>

<!-- Verbesserte Info-Leiste -->
<div class="absolute top-4 right-4 z-10 bg-white text-sm text-gray-800 shadow-lg rounded px-4 py-2 max-w-xs">
    {#if zoomLevel < 14}
        <p class="flex items-center gap-2">
            <span>🔍</span>
            <span>Bitte weiter reinzoomen, um Bänke anzuzeigen (Zoom ≥ 14)</span>
        </p>
        <p class="text-xs text-gray-500 mt-1">Aktueller Zoom: {zoomLevel}</p>
    {:else if loading}
        <p class="flex items-center gap-2">
            <span>⏳</span>
            <span>Lade Bänke in der Umgebung…</span>
        </p>
    {:else}
        <p class="flex items-center gap-2">
            <span>✅</span>
            <span>Alle sichtbaren Bänke geladen</span>
        </p>
        <p class="text-xs text-gray-500 mt-1">
            {loadedBenchesCount} Bänke gefunden
        </p>
    {/if}
</div>

<!-- Zusätzliche Kontrollleiste -->
<div class="absolute bottom-4 right-4 z-10 bg-white shadow-lg rounded px-3 py-2">
    <button 
        on:click={clearAllData}
        class="text-xs bg-red-500 text-white px-2 py-1 rounded hover:bg-red-600 transition-colors"
        title="Cache leeren und neu laden"
    >
        🔄 Neu laden
    </button>
</div>

<style>
    .leaflet-tile {
        image-rendering: pixelated;
        will-change: transform;
    }
    
    /* Verbesserte Popup-Styles */
    :global(.leaflet-popup-content) {
        font-size: 12px;
        line-height: 1.4;
    }
</style>
