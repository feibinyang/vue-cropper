<template>
  <div id="app">
    <img alt="Vue logo" src="./assets/logo.png">
    <!-- <HelloWorld msg="Welcome to Your Vue.js App"/> -->
    <Cropper ref="cropper" url="http://i3.baidu.com/it/u=3841678517,475667367&fm=207"
        :width="400" :height="400" :ratio="1/1" :area=".5" :clip-data="{}" />
  </div>
</template>

<script>
// import HelloWorld from './components/HelloWorld.vue'
import Cropper from './components/Cropper.vue';

export default {
  name: 'app',
  components: {
    // HelloWorld
    Cropper
  },
  mounted() {
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
.cropper-container {
  margin: 30px auto;
}
</style>
