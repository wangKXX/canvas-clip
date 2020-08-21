<template>
  <div class="clip-wrap">
    <div class="clip-menu">
      <span class="show-part">
        <span class="txt">宽度：</span><input v-model="clipWidth"/>
        <span class="txt">高度：</span><input v-model="clipHeight"/>
      </span>
      <span>
        <span class="txt btn" @click="handleClick">裁剪</span>
      </span>
    </div>
    <div class="clip" ref="clip">
      <canvas ref="canvas"></canvas>
      <clip-mask :width="imgWidth" :height="imgHeight" :initClipHeight="clipHeight" :initClipWidth="clipWidth" @widthHeightChange="widthHeightChange"/>
    </div>
    <img :src="previewImage" v-if="previewImage" class="priview"/>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from "vue-property-decorator";
import clipMask from "./mask.vue";

@Component({
  components: {
    clipMask,
  },
})
export default class Clip extends Vue {
  public $refs!: {
    canvas: HTMLCanvasElement,
  };

  @Prop() private src!: string;
  @Prop({ default: 500 }) private height!: number;

  private imgWidth: number = 0;
  private imgHeight: number = 0;
  private canvasCtx!: any;
  private imageType!: string;
  private previewImage: any = "";
  private clipHeight: number = 200;
  private clipWidth: number = 200;
  private top: number = 0;
  private left: number = 0;

  public created() {
    const img = new Image();
    img.onload = (e: any) => {
      const { width, height, type } = e.target;
      this.imageType = type;
      const scale: any = this.scale(width, height);
      this.setCanvasWH(scale);
      // this.initCanvasCtx();
      this.drawImageToCanvas(img);
    };
    img.src = this.src;
  }

  private setCanvasWH(scale: any) {
    const { canvas } = this.$refs;
    canvas.width = scale.width;
    canvas.height = scale.height;
    this.imgWidth = scale.width;
    this.imgHeight = scale.height;
    this.initCanvasCtx();
  }

  private initCanvasCtx() {
    const { canvas } = this.$refs;
    this.canvasCtx = canvas.getContext("2d");
  }

  private drawImageToCanvas(img: any) {
    const { imgWidth, imgHeight } = this;
    this.canvasCtx.drawImage(img, 0, 0, imgWidth, imgHeight);
  }


  private scale(w: number, h: number): object {
    const { height = 500 } = this;
    const sWidth = (w * height) / h;
    return { width: sWidth, height };
  }

  private handleClick(): void {
    const { left, top, clipWidth, clipHeight } = this;
    this.clipCanvasToImage(left, top, clipWidth, clipHeight);
  }

  private clipCanvasToImage(x: number, y: number, width: number, height: number): void {
    const imageData = this.canvasCtx.getImageData(x, y, width, height);
    const tempCanvas = document.createElement("canvas");
    tempCanvas.width = width;
    tempCanvas.height = height;
    const ctx: any = tempCanvas.getContext("2d");
    ctx.putImageData(imageData, 0, 0, 0, 0, width, height);
    this.previewImage = tempCanvas.toDataURL(this.imageType, 1.0);
  }

  private widthHeightChange(obj: any) {
    const { width, height, top, left } = obj;
    Object.assign(this, {
      clipWidth: width,
      clipHeight: height,
      top,
      left,
    });
  }
}
</script>
