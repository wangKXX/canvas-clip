<template>
  <div id="app">
    <clip
      :src="src"
      :clipHeight="100"
      :clipWidth="100"
      :height="400"
      :showPreview="true"
      :showControls="true"
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
  private src: any = "https://cdn.glitch.com/4c9ebeb9-8b9a-4adc-ad0a-238d9ae00bb5%2Fmdn_logo-only_color.svg?1535749917189";
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
