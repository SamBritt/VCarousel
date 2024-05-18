<script setup lang="ts">
import VCarousel from './VCarousel.vue'
</script>

<template>
  <div class="wrapper">
    <VCarousel class="carousel">
      <div
        v-for="n of 12"
        class="carousel-item"></div>
    </VCarousel>
  </div>
</template>

<style scoped lang="scss">
.wrapper {
  height: inherit;
}
.carousel {
  &-item {
    width: 20rem;
    height: 10rem;
    background: rgb(87, 87, 192);
    margin: 0 0.25rem;

    &:first-child {
      margin-left: 0;
    }

    &:last-child {
      margin-right: 0;
    }
  }
}
</style>
