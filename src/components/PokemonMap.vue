<template>
<div>
    <div ref="mapContainer" class="map-container"></div>
</div>
</template>

<script setup lang="ts">
import {
    onMounted,
    ref
} from 'vue';
import mapboxgl from 'mapbox-gl';

const mapContainer = ref < HTMLDivElement | null > (null);
mapboxgl.accessToken =
    import.meta.env.VITE_MAPBOX_TOKEN;

const pokemons = [{
        id: 25,
        name: 'Pikachu',
        lat: 35.6895,
        lng: 139.6917
    },
    {
        id: 4,
        name: 'Charmander',
        lat: 34.0522,
        lng: -118.2437
    },
    {
        id: 7,
        name: 'Squirtle',
        lat: 51.5074,
        lng: -0.1278
    },
];

onMounted(() => {
   if (!mapContainer.value) return;
  const map = new mapboxgl.Map({
    container: mapContainer.value,
    style: 'mapbox://styles/mapbox/streets-v11',
    center: [0, 30],
    zoom: 2,
  });
  map.on('load', () => map.resize());

    pokemons.forEach(async (pokemon) => {
        const res = await fetch(`https://pokeapi.co/api/v2/pokemon/${pokemon.id}`);
        const data = await res.json();
        const spriteUrl = data.sprites.front_default;

        if (!spriteUrl) return;

        const el = document.createElement('div');
        el.className = 'pokemon-marker';
        el.style.backgroundImage = `url(${spriteUrl})`;
        el.style.width = '64px';
        el.style.height = '64px';
        el.style.backgroundSize = 'contain';
        el.style.backgroundRepeat = 'no-repeat';
        el.style.backgroundPosition = 'center';

        new mapboxgl.Marker(el)
            .setLngLat([pokemon.lng, pokemon.lat])
            .setPopup(new mapboxgl.Popup().setText(pokemon.name))
            .addTo(map);
    });
});
</script>

<style scoped>
.map-container {
    width: 100vw;
    height: 100vh;
    background: lightgray;
    /* helps verify it's rendering */
}

.pokemon-marker {
    border-radius: 50%;
    background-color: rgba(255, 255, 255, 0.8);
}
</style>
