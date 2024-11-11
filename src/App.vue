<template>
  <div id="app">
    <BitcoinChart />
  </div>
</template>

<script>
import BitcoinChart from './components/BitcoinChart.vue'

export default {
  name: 'App',
  components: {
    BitcoinChart
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  min-height: 100vh;
  width: 100%;
  margin: 0;
  padding: 20px;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  align-items: center;
}

html, body {
  margin: 0;
  padding: 0;
  height: 100%;
  width: 100%;
}

.wallet-container {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 20px;
  width: 100%;
  max-width: 1200px;
}
</style>
