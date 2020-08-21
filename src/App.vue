<template>
  <div id="app">
    <clip
      :src="src"
    />
  </div>
</template>

<script lang="ts">
import { Component, Vue } from "vue-property-decorator";
import clip from "@/plugin/index.vue";

@Component({
  components: {
    clip,
  },
})
export default class App extends Vue {
  private src: any = require("./assets/1.jpeg");
}
</script>

<style lang="scss" scoped>
#app{
  width: 100%;
  height: 100%;
  box-sizing: border-box;
  margin: 0px auto;
  padding: 20px;
}
</style>
