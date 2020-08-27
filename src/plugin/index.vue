<template>
  <div class="clip-wrap">
    <div class="clip-menu">
      <span class="show-part" v-if="showControls">
        <span class="txt">宽度：</span><input v-model="maskClipWidth" />
        <span class="txt">高度：</span><input v-model="maskClipHeight" />
        <span class="txt">旋转：</span
        ><input v-model="transform" type="range" min="0" max="180" step="1" />
      </span>
      <span>
        <span class="txt btn" @click="handleClick">裁剪</span>
      </span>
    </div>
    <div
      class="clip"
      ref="clip"
      @mousewheel="handleMouseWheel"
      @mousedown.stop="handleMouseDown"
      @mousemove.stop="handleMouseMove"
      @mouseup.stop="handleMouseUp"
      @drop.stop.prevent="dropImage"
      @dragleave.stop.prevent
      @dragover.stop.prevent
      @dragenter.stop.prevent
      @dragend.stop.prevent
      @dragstart.stop.prevent
    >
      <canvas ref="canvas"></canvas>
      <clip-mask
        :width="imgWidth"
        :height="imgHeight"
        :initClipHeight="+clipHeight"
        :initClipWidth="+clipWidth"
        @widthHeightChange="widthHeightChange"
        @sendSelf="getChild"
        ref="clip"
      />
    </div>
    <div class="resoult-part" v-if="previewImage && showPreview">
      <img :src="previewImage" class="priview" />
      <a :href="previewImage" :download="imageName">下载</a>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue, Watch } from "vue-property-decorator";
import clipMask from "./mask.vue";

@Component({
  components: {
    clipMask,
  },
})
export default class Clip extends Vue {
  public $refs!: {
    canvas: HTMLCanvasElement;
    clip: any;
  };

  public imgWidth: number = 0;
  public imgHeight: number = 0;
  public canvasCtx!: any;
  public imageType!: string;
  public imageName!: string;
  public previewImage: any = "";
  public maskClipHeight: number = 200;
  public maskClipWidth: number = 200;
  public transform: number = 0;
  public top: number = 0;
  public left: number = 0;
  public img!: any;
  public scaleNumber: number = 1;
  public isImgCanMove: boolean = false;
  public startX!: number;
  public startY!: number;
  public isMoving: boolean = false;
  public clip!: any;

  @Prop() public src!: string;
  @Prop({ default: 500 }) public height!: number;
  @Prop({ default: 200 }) public clipHeight!: number;
  @Prop({ default: 200 }) public clipWidth!: number;
  @Prop({ default: true }) public showPreview!: boolean;
  @Prop({ default: true }) public showControls!: boolean;

  public clipImage() {
    const { left, top, maskClipWidth, maskClipHeight } = this;
    this.clipCanvasToImage(left, top, maskClipWidth, maskClipHeight);
  }

  @Watch("transform")
  private transformChange(value: number) {
    const { imgWidth, imgHeight } = this;
    const { canvas } = this.$refs;
    this.canvasCtx.clearRect(-1, -1, imgWidth + 4, imgHeight + 4);
    this.canvasCtx.translate(imgWidth / 2, imgHeight / 2);
    this.canvasCtx.rotate(value * Math.PI / 180);
    this.canvasCtx.translate(-imgWidth / 2, -imgHeight / 2);
    this.drawImageToCanvas(this.img);
    this.canvasCtx.translate(imgWidth / 2, imgHeight / 2);
    this.canvasCtx.rotate(-value * Math.PI / 180);
    this.canvasCtx.translate(-imgWidth / 2, -imgHeight / 2);
  }

  private getChild(clip: any) {
    this.clip = clip;
  }

  private created() {
    this.setImageData(this.src);
  }

  private setImageData(src: string) {
    this.img = new Image();
    this.img.onload = (e: any) => {
      const { width, height, type, name } = e.target;
      this.imageType = type;
      this.imageName = name;
      const scale: any = this.scale(width, height);
      this.setCanvasWH(scale);
      this.drawImageToCanvas(this.img);
    };
    this.img.src = src;
  }

  private dropImage(event: any) {
    const fileData = event.dataTransfer.files;
    const [file] = fileData;
    const { type } = file;
    if (/^image/.test(type)) {
      const fileReader = new FileReader();
      fileReader.onload = (e: any) => {
        const {
          target: { result },
        } = e;
        this.setImageData(result);
      };
      fileReader.readAsDataURL(file);
    }
  }

  private handleMouseWheel(e: any) {
    const { wheelDelta } = e;
    const { imgWidth, imgHeight, scaleNumber } = this;
    this.canvasCtx.clearRect(0, 0, imgWidth + 4, imgHeight + 4);
    if (wheelDelta > 0) {
      // 放大
      this.scaleNumber = scaleNumber + 1;
      this.isImgCanMove = this.scaleNumber > 1;
    } else {
      this.scaleNumber = scaleNumber - 1;
    }
    const scale = 1 + this.scaleNumber / 10;
    this.canvasCtx.setTransform(
      scale,
      0,
      0,
      scale,
      (imgWidth - imgWidth * scale) / 2,
      (imgHeight - imgHeight * scale) / 2,
    );
    this.drawImageToCanvas(this.img);
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

  private drawImageToCanvas(img: any, x?: number, y?: number) {
    const { imgWidth, imgHeight } = this;
    this.canvasCtx.drawImage(img, x || 0, y || 0, imgWidth, imgHeight);
  }

  private scale(w: number, h: number): object {
    const { height = 500 } = this;
    const sWidth = (w * height) / h;
    return { width: sWidth, height };
  }

  private handleClick(): void {
    const { left, top, maskClipWidth, maskClipHeight } = this;
    this.clipCanvasToImage(left, top, maskClipWidth, maskClipHeight);
  }

  private clipCanvasToImage(
    x: number,
    y: number,
    width: number,
    height: number,
  ): void {
    const imageData = this.canvasCtx.getImageData(x, y, width, height);
    const tempCanvas = document.createElement("canvas");
    tempCanvas.width = width;
    tempCanvas.height = height;
    const ctx: any = tempCanvas.getContext("2d");
    ctx.putImageData(imageData, 0, 0, 0, 0, width, height);
    this.previewImage = tempCanvas.toDataURL(this.imageType, 1.0);
    this.$emit("clipCallBack", this.previewImage);
  }

  private widthHeightChange(obj: any) {
    const { width, height, top, left } = obj;
    Object.assign(this, {
      maskClipWidth: width,
      maskClipHeight: height,
      top,
      left,
    });
  }

  private handleMouseDown(e: MouseEvent) {
    const { offsetX, offsetY } = e;
    this.startX = offsetX;
    this.startY = offsetY;
    this.isMoving = true;
  }

  private handleMouseMove(e: MouseEvent) {
    const {
      isImgCanMove,
      startX,
      startY,
      imgWidth,
      imgHeight,
      isMoving,
    } = this;
    const { offsetX, offsetY } = e;
    if (isImgCanMove && isMoving) {
      const isInMenu = this.clip.isInMenuPart(offsetX, offsetY);
      if (isInMenu) {
        return false;
      }
      const offsetW = offsetX - startX;
      const offsetH = offsetY - startY;
      this.canvasCtx.clearRect(-1, -1, imgWidth + 4, imgHeight + 4);
      this.canvasCtx.translate(offsetW, offsetH);
      this.drawImageToCanvas(this.img);
      this.startX = offsetX;
      this.startY = offsetY;
    }
  }

  private handleMouseUp(e: MouseEvent) {
    this.isMoving = false;
  }
}
</script>
