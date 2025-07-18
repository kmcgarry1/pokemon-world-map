<template>
  <div ref="mapContainer" class="map-container"></div>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue';
import mapboxgl from 'mapbox-gl';

const mapContainer = ref<HTMLDivElement | null>(null);
mapboxgl.accessToken = import.meta.env.VITE_MAPBOX_TOKEN;

// --- 1. Load last known position from localStorage (if any) ---
interface Position { longitude: number; latitude: number; }
let initialCenter: [number, number] = [0, 30];  // default fallback

const saved = localStorage.getItem('lastPosition');
if (saved) {
  try {
    const { longitude, latitude } = JSON.parse(saved) as Position;
    initialCenter = [longitude, latitude];
  } catch {
    // ignore parse errors
  }
}

// Your Pokémon list
const pokemons = [
  { id: 25, name: 'Pikachu', lat: 35.6895, lng: 139.6917 },
  { id: 4,  name: 'Charmander', lat: 34.0522, lng: -118.2437 },
  { id: 7,  name: 'Squirtle',  lat: 51.5074, lng: -0.1278 },
];

onMounted(() => {
  if (!mapContainer.value || !navigator.geolocation) return;

  // --- 2. Initialize Mapbox map with saved or fallback center ---
  const map = new mapboxgl.Map({
    container: mapContainer.value,
    style: 'mapbox://styles/mapbox/streets-v11',
    center: initialCenter,
    zoom: 18,
    minZoom: 18,
    maxZoom: 20,
  });

  map.on('load', () => {
    map.resize();

    // Add Mapbox’s geolocate control (optional, for UI)
    const geolocate = new mapboxgl.GeolocateControl({
      positionOptions: { enableHighAccuracy: true },
      trackUserLocation: true,
      showAccuracyCircle: false,
      fitBoundsOptions: { maxZoom: 18 },
    });
    map.addControl(geolocate, 'top-left');
  });

  // --- 3. Watch the user’s position and update map & localStorage ---
  navigator.geolocation.watchPosition(
    ({ coords }) => {
      const { longitude, latitude } = coords;

      // Recenter map
      map.setCenter([longitude, latitude]);

      // Persist for next session
      localStorage.setItem(
        'lastPosition',
        JSON.stringify({ longitude, latitude })
      );
    },
    (err) => {
      console.warn('Geolocation error:', err.message);
    },
    { enableHighAccuracy: true, maximumAge: 0, timeout: 5000 }
  );

  // --- 4. Add your sprite markers ---
  pokemons.forEach(async (pokemon) => {
    try {
      const res = await fetch(`https://pokeapi.co/api/v2/pokemon/${pokemon.id}`);
      const data = await res.json();
      const spriteUrl = data.sprites.front_default;
      if (!spriteUrl) return;

      const el = document.createElement('div');
      el.className = 'pokemon-marker';
      el.style.width = '48px';
      el.style.height = '48px';
      el.style.backgroundImage = `url(${spriteUrl})`;
      el.style.backgroundSize = 'contain';
      el.style.backgroundRepeat = 'no-repeat';
      el.style.backgroundPosition = 'center';
      el.style.cursor = 'pointer';

      new mapboxgl.Marker(el)
        .setLngLat([pokemon.lng, pokemon.lat])
        .setPopup(new mapboxgl.Popup({ offset: 25 }).setText(pokemon.name))
        .addTo(map);
    } catch (e) {
      console.error(`Error loading ${pokemon.name}:`, e);
    }
  });
});
</script>

<style scoped>
.map-container {
  width: 100vw;
  height: 100vh;
}

.pokemon-marker {
  border-radius: 50%;
  background-color: rgba(255, 255, 255, 0.8);
}
</style>
