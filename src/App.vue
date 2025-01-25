<template>
  <div class="app">
    <h1>汽车仪表盘</h1>
    <Speedometer />
  </div>
</template>

<script setup>
import Speedometer from './components/Speedometer.vue'
</script>

<style>
body {
  margin: 0;
  padding: 20px;
  font-family: Arial, sans-serif;
}

.app {
  max-width: 800px;
  margin: 0 auto;
  text-align: center;
}

h1 {
  color: #333;
  margin-bottom: 30px;
}
</style>
