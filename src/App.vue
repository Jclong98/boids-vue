<script setup>
import Boids from './components/Boids.vue'
</script>

<template>
  <Boids />
  <div class="attribution">
    <p>Made by <a href="https://jacoblong.dev">Jacob Long</a></p>
    <p><a href="https://github.com/jclong98/boids-vue">Source</a></p>
  </div>
</template>

<style>
html,
body {
  margin: 0;
  padding: 0;
}

* {
  box-sizing: border-box;
}

.attribution {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 0.5rem;
  line-height: 0.5;
  color: white;
  text-align: center;
}

.attribution a {
  color: tomato;
}
</style>
