<template>
  <div id="app" class="light-mode">
    <Game />
  </div>
</template>

<script>
import Game from './components/Game.vue';

export default {
  name: 'App',
  components: {
    Game
  }
};
</script>

<style>
.light-mode {
  background-color: #fff;
  color: #000;
}
</style>