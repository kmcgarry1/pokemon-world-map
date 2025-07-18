<template>
  <div ref="mapContainer" class="map-container"></div>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue';
import mapboxgl from 'mapbox-gl';

mapboxgl.accessToken = import.meta.env.VITE_MAPBOX_TOKEN;
const mapContainer = ref<HTMLDivElement | null>(null);

// — your species list —
const pokemons = [
  { id: 25, name: 'Pikachu' },
  { id: 4,  name: 'Charmander' },
  { id: 7,  name: 'Squirtle' },
];

// radius around user to spawn (km)
const SPAWN_RADIUS_KM = 0.1;

// helper: pick random species
function randomPokemon() {
  return pokemons[Math.floor(Math.random() * pokemons.length)];
}

// helper: random point within radius around (lat, lon)
function randomPointInRadius(
  lon: number,
  lat: number,
  radiusKm: number
): [number, number] {
  const R = 6371; // earth radius km
  // distance from center (km), random with uniform area
  const d = Math.sqrt(Math.random()) * radiusKm;
  const θ = Math.random() * 2 * Math.PI;

  // offset in radians
  const δ = d / R;

  // from: https://stackoverflow.com/a/74707709
  const φ1 = (lat * Math.PI) / 180;
  const λ1 = (lon * Math.PI) / 180;

  const φ2 = Math.asin(
    Math.sin(φ1) * Math.cos(δ) +
      Math.cos(φ1) * Math.sin(δ) * Math.cos(θ)
  );
  const λ2 =
    λ1 +
    Math.atan2(
      Math.sin(θ) * Math.sin(δ) * Math.cos(φ1),
      Math.cos(δ) - Math.sin(φ1) * Math.sin(φ2)
    );

  return [(λ2 * 180) / Math.PI, (φ2 * 180) / Math.PI];
}

// track current user coords
interface Position { longitude: number; latitude: number; }
let currentPos: Position = { longitude: 0, latitude: 30 };

// load saved position
const saved = localStorage.getItem('lastPosition');
if (saved) {
  try {
    currentPos = JSON.parse(saved);
  } catch {}
}

onMounted(() => {
  if (!mapContainer.value || !navigator.geolocation) return;

  // init map at last known or default
  const map = new mapboxgl.Map({
    container: mapContainer.value,
    style: 'mapbox://styles/kevinmcg/cmd8pc6rf00w001s9bbnt0ahn',
    center: [currentPos.longitude, currentPos.latitude],
    zoom: 18,
    minZoom: 18,
    maxZoom: 20,
  });

  map.on('load', () => {
    map.resize();

    // geolocate control
    const geoCtrl = new mapboxgl.GeolocateControl({
      positionOptions: { enableHighAccuracy: true },
      trackUserLocation: true,
      showAccuracyCircle: false,
      fitBoundsOptions: { maxZoom: 18 },
    });
    map.addControl(geoCtrl, 'top-left');
    geoCtrl.trigger();

    // GeoJSON source & layer for wild spawns
    map.addSource('wild-spawns', {
      type: 'geojson',
      data: { type: 'FeatureCollection', features: [] },
    });
    map.addLayer({
      id: 'wild-spawn-markers',
      type: 'symbol',
      source: 'wild-spawns',
      layout: {
        'icon-image': ['concat', ['get', 'spriteId'], '-icon'],
        'icon-size': 1,
        'icon-allow-overlap': true,
      },
    });

    // preload images into sprite atlas
    pokemons.forEach(async (p) => {
      const res = await fetch(`https://pokeapi.co/api/v2/pokemon/${p.id}`);
      const data = await res.json();
      const url = data.sprites.front_default;
      if (url) {
        map.loadImage(url, (e, img) => {
          if (img) map.addImage(`${p.id}-icon`, img);
        });
      }
    });
  });

  // watch user location
  navigator.geolocation.watchPosition(
    ({ coords }) => {
      currentPos = {
        longitude: coords.longitude,
        latitude: coords.latitude,
      };
      map.setCenter([coords.longitude, coords.latitude]);
      localStorage.setItem('lastPosition', JSON.stringify(currentPos));
    },
    (err) => console.warn('Geolocation error:', err.message),
    { enableHighAccuracy: true, maximumAge: 0, timeout: 5000 }
  );

  // spawn loop
  setInterval(() => {
    // random point within radius
    const [lon, lat] = randomPointInRadius(
      currentPos.longitude,
      currentPos.latitude,
      SPAWN_RADIUS_KM
    );

    // ensure in viewport
    const b = map.getBounds();
    if (lon < b.getWest() || lon > b.getEast() ||
        lat < b.getSouth() || lat > b.getNorth()) {
      return;
    }

    // avoid water
    const p = map.project([lon, lat]);
    const feats = map.queryRenderedFeatures(p, {
      layers: ['landcover']
    });
    if (feats.some(f => f.properties?.class === 'water')) {
      return;
    }

    // pick a random Pokémon
    const { id } = randomPokemon();

    // add to GeoJSON source
    const src = map.getSource('wild-spawns') as mapboxgl.GeoJSONSource;
    const fc = (src as any)._data as GeoJSON.FeatureCollection;
    const feature: GeoJSON.Feature = {
      type: 'Feature',
      properties: { spriteId: id },
      geometry: {
        type: 'Point',
        coordinates: [lon, lat],
      },
    };
    src.setData({
      ...fc,
      features: fc.features.concat(feature),
    });
  }, 5000);
});
</script>

<style scoped>
.map-container {
  width: 100vw;
  height: 100vh;
}
</style>
