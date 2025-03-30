<template>
  <img alt="Vue logo" src="./assets/karuta.png">
  <ImasKaruta msg="Welcome to Your Vue.js App"/>
</template>

<script>
import ImasKaruta from './components/ImasKaruta.vue'

export default {
  name: 'App',
  components: {
    ImasKaruta
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
  margin-top: 60px;
}

/* 画像をレスポンシブにするスタイル */
img {
  max-width: 20%;
  height: auto;
}
</style>
