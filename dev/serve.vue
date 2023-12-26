<script>
import Vue from 'vue';
import NbmapModelZk from '@/nbmap-model-zk.vue';

export default Vue.extend({
  name: 'ServeDev',
  components: {
    NbmapModelZk
  }
});
</script>

<template>
  <div id="app">
    <nbmap-model-zk />
  </div>
</template>

<style>
body,html,#app{
  margin: 0;
  padding: 0;
  font-size: 0;
  height: 100%;
  width: 100%;
}
</style>
