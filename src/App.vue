<template>
  <div class="app-container">
    <PokemonMap />
  </div>
</template>

<script setup lang="ts">
import PokemonMap from './components/PokemonMap.vue';
</script>

<style>
html,
body,
#app {
  margin: 0;
  padding: 0;
  height: 100%;
  width: 100%;
}

.app-container {
  height: 100%;
  width: 100%;
}
</style>
