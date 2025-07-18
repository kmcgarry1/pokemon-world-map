<template>
  <div ref="mapContainer" class="map-container"></div>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue';
import mapboxgl from 'mapbox-gl';

const mapContainer = ref<HTMLDivElement | null>(null);
mapboxgl.accessToken = import.meta.env.VITE_MAPBOX_TOKEN;

const pokemons = [
  { id: 25, name: 'Pikachu', lat: 35.6895, lng: 139.6917 },
  { id: 4,  name: 'Charmander', lat: 34.0522, lng: -118.2437 },
  { id: 7,  name: 'Squirtle',  lat: 51.5074, lng: -0.1278 },
];

onMounted(() => {
  if (!mapContainer.value || !navigator.geolocation) return;

  navigator.geolocation.getCurrentPosition(
    ({ coords }) => {
      const { longitude, latitude } = coords;

      // 1️⃣ Initialize Mapbox map at user location
      const map = new mapboxgl.Map({
        container: mapContainer.value as HTMLElement,
        style: 'mapbox://styles/mapbox/streets-v11',
        center: [longitude, latitude],
        zoom: 18,
        minZoom: 16,
        maxZoom: 20,
      });

      map.on('load', () => {
        map.resize();

        // 2️⃣ Add GeolocateControl to keep following user
        const geolocate = new mapboxgl.GeolocateControl({
          positionOptions: { enableHighAccuracy: true },
          trackUserLocation: true,
          showAccuracyCircle: false,
          fitBoundsOptions: { maxZoom: 18 },
        });
        map.addControl(geolocate, 'top-left');
        geolocate.trigger(); // immediately recenter
      });

      // 3️⃣ Place Pokémon sprite markers
      pokemons.forEach(async (pokemon) => {
        try {
          const res = await fetch(`https://pokeapi.co/api/v2/pokemon/${pokemon.id}`);
          const data = await res.json();
          const spriteUrl = data.sprites.front_default;
          if (!spriteUrl) return;

          const el = document.createElement('div');
          el.className = 'pokemon-marker';
          el.style.backgroundImage = `url(${spriteUrl})`;
          el.style.width = '48px';
          el.style.height = '48px';
          el.style.backgroundSize = 'contain';
          el.style.backgroundRepeat = 'no-repeat';
          el.style.backgroundPosition = 'center';
          el.style.cursor = 'pointer';

          new mapboxgl.Marker(el)
            .setLngLat([pokemon.lng, pokemon.lat])
            .setPopup(new mapboxgl.Popup({ offset: 25 }).setText(pokemon.name))
            .addTo(map);
        } catch (e) {
          console.error(`Error loading ${pokemon.name}`, e);
        }
      });
    },
    (err) => {
      console.error('Geolocation failed:', err.message);
      // Fallback: initialize map at a default center
      const map = new mapboxgl.Map({
        container: mapContainer.value as HTMLElement,
        style: 'mapbox://styles/mapbox/streets-v11',
        center: [139.6917, 35.6895], // Default to Tokyo
        zoom: 16,
      });
    },
    { enableHighAccuracy: true, timeout: 5000 }
  );
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
