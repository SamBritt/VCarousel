<script setup lang="ts">
import Home from './components/Home.vue'
</script>

<template>
  <header>
    <div class="wrapper"></div>
  </header>

  <main class="main">
    <Home />
  </main>
</template>

<style scoped lang="scss">
.main {
  height: inherit;
  padding: 2rem;
}
</style>
