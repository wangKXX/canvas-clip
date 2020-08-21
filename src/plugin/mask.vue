<template>
  <canvas
    ref="maskCanvas"
    class="mask-canvas"
    @mousedown="handleMouseDown"
    @mouseup="handleMouseUp"
    @mousemove="handleMouseMove"
  ></canvas>
</template>

<script lang="ts">
import { Component, Prop, Vue, Watch } from "vue-property-decorator";

@Component
export default class ClipMask extends Vue {
  public $refs!: {
    mask: HTMLDivElement;
    maskCanvas: HTMLCanvasElement;
  };

  @Prop() private width!: number; // 图片初始宽度
  @Prop() private height!: number; // 图片初始高度
  @Prop() private initClipWidth!: number; // 裁剪区初始宽度
  @Prop() private initClipHeight!: number; // 裁剪区初始高度

  private left: number = 1;
  private top: number = 1;
  private startX!: number;
  private startY!: number;
  private offsetLeft!: number;
  private offsetTop!: number;
  private startPointX!: number;
  private startPointY!: number;
  private moving: boolean = false;
  private clipHeight: number = 300;
  private clipWidth: number = 300;
  private timer!: any;
  private selectPoint!: any;
  private maskCanvasCtx!: any;
  private offset: number = 0;
  private maskCanvasData!: any;
  private borderPath!: any;
  private menuPoints!: object[];

  @Watch("clipHeight")
  @Watch("clipWidth")
  @Watch("top")
  @Watch("left")
  private onWidthChange(value: any) {
    this.$emit("widthHeightChange", {
      width: this.clipWidth,
      height: this.clipHeight,
      top: this.top,
      left: this.left,
    });
  }

  @Watch("width")
  @Watch("height")
  private widthChange(value: any) {
    this.initMaskCanvas();
  }

  private created() {
    document.addEventListener("mouseup", this.resetStatus);
    Object.assign(this, {
      clipHeight: this.initClipHeight || 300,
      clipWidth: this.initClipWidth || 300,
    });
  }

  private mounted() {
    this.initMaskCanvas();
  }

  private destroyed() {
    document.removeEventListener("mouseup", this.resetStatus);
  }

  private initMaskCanvas() {
    const { maskCanvas } = this.$refs;
    const { width, height } = this;
    maskCanvas.width = width;
    maskCanvas.height = height;
    this.maskCanvasCtx = maskCanvas.getContext("2d");
    this.maskCanvasClear();
    this.dashBorderLinemarch();
    this.setInitTopLeft();
  }

  private setInitTopLeft() {
    const { clipWidth, clipHeight, width, height } = this;
    this.left = (width - clipWidth) / 2;
    this.top = (height - clipHeight) / 2;
  }

  private maskCanvasClear() {
    const { width, height } = this;
    this.maskCanvasCtx.clearRect(0, 0, width, height);
    this.maskCanvasCtx.fillStyle = "rgba(255, 255, 255, .6)";
    this.maskCanvasCtx.fillRect(0, 0, width, width);
  }

  private setClipBorder() {
    const { clipWidth, clipHeight, top, left, maskCanvasCtx, offset } = this;
    this.maskCanvasClear();
    maskCanvasCtx.beginPath();
    maskCanvasCtx.setLineDash([4, 2]);
    maskCanvasCtx.lineDashOffset = -offset;
    maskCanvasCtx.lineWidth = 1;
    maskCanvasCtx.strokeStyle = "#666";
    maskCanvasCtx.rect(left, top, clipWidth, clipHeight);
    maskCanvasCtx.closePath();
    maskCanvasCtx.stroke();
    maskCanvasCtx.restore();
    this.borderPath = maskCanvasCtx;
    this.setClipMask();
    this.setMenuPoints();
  }

  private setDoMenuPoint(x: number, y: number) {
    const { maskCanvasCtx } = this;
    maskCanvasCtx.beginPath();
    maskCanvasCtx.fillStyle = "#666";
    maskCanvasCtx.rect(x, y, 9, 9);
    maskCanvasCtx.closePath();
    maskCanvasCtx.fill();
    maskCanvasCtx.restore();
  }

  private setMenuPoints() {
    this.menuPoints = [];
    const { clipWidth, clipHeight, top, left } = this;
    const topLeft = { x: left, y: top, position: "topLeft" };
    const topLeftMiddle = {
      x: left + clipWidth / 2,
      y: top,
      position: "topLeftMiddle",
    };
    const topRight = { x: left + clipWidth, y: top, position: "topRight" };
    const topBottomLeftMiddle = {
      x: left,
      y: top + clipHeight / 2,
      position: "topBottomLeftMiddle",
    };
    const topBottomRightMiddle = {
      x: left + clipWidth,
      y: top + clipHeight / 2,
      position: "topBottomRightMiddle",
    };
    const bottomLeft = { x: left, y: top + clipHeight, position: "bottomLeft" };
    const bottomLeftMiddle = {
      x: left + clipWidth / 2,
      y: top + clipHeight,
      position: "bottomLeftMiddle",
    };
    const bottomRight = {
      x: left + clipWidth,
      y: top + clipHeight,
      position: "bottomRight",
    };
    this.menuPoints.push(
      topLeft,
      topLeftMiddle,
      topRight,
      topBottomLeftMiddle,
      topBottomRightMiddle,
      bottomLeft,
      bottomLeftMiddle,
      bottomRight,
    );
    this.menuPoints.forEach(({ x, y }: any) =>
      this.setDoMenuPoint(x - 4, y - 4),
    );
  }

  private setCursor(x: number, y: number, target: HTMLCanvasElement): void {
    const { clipWidth, clipHeight, top, left } = this;
    const path = this.getPath2DRect(left, top, clipWidth, clipHeight);
    if (this.isPointInClip(x, y, path)) {
      target.style.cursor = "move";
    } else {
      target.style.cursor = "pointer";
    }
  }

  private dashBorderLinemarch() {
    this.offset = this.offset + 1;
    if (this.offset > 16) {
      this.offset = 0;
    }
    this.setClipBorder();
    setTimeout(this.dashBorderLinemarch, 80);
  }

  /**
   * 裁剪区域透明设置
   */
  private setClipMask() {
    const { clipWidth, clipHeight, top, left } = this;
    this.maskCanvasCtx.clearRect(
      left + 1,
      top + 1,
      clipWidth - 2,
      clipHeight - 2,
    );
  }

  private getPath2DRect(
    x: number,
    y: number,
    width: number,
    height: number,
  ): Path2D {
    // M10 10 h 40 v 40 h -40 Z
    return new Path2D(`M${x} ${y} h ${width} v ${height} h -${width} Z`);
  }

  // private getPath2DArc(x: number, y: number): Path2D {
  //   /**
  //    * M cx, cy
  //       m -r, 0
  //       a r,r 0 1,0 (r * 2),0
  //       a r,r 0 1,0 -(r * 2),0

  //    */
  //   return new Path2D(
  //     `M ${x}, ${y} m -5, 0 a 5,5 0 1,0 (${5} * 2),0 a 5, 5 0 1,0 (${5} * 2),0`
  //   );
  // }

  /**
   * 鼠标是否在裁剪区域
   */
  private isPointInClip(x: number, y: number, path: Path2D): boolean {
    const { maskCanvasCtx, top, left, clipWidth, clipHeight } = this;
    return maskCanvasCtx.isPointInPath(path, x, y);
  }

  private isPointInClipBorder(x: number, y: number): boolean {
    const { maskCanvasCtx } = this;
    return maskCanvasCtx.isPointInStroke(x, y);
  }

  private getLocation(x: number, y: number) {
    const { maskCanvas } = this.$refs;
    const bbox = maskCanvas.getBoundingClientRect();
    return {
      x: x - bbox.left,
      y: y - bbox.top,
    };
  }

  private saveMaskCanvasData(): void {
    this.maskCanvasData = this.maskCanvasCtx.getImageData(
      0,
      0,
      this.width,
      this.height,
    );
  }

  private resetCanvasData(): void {
    const { maskCanvasData, width, height } = this;
    if (maskCanvasData) {
      this.maskCanvasCtx.putImageData(
        maskCanvasData,
        0,
        0,
        0,
        0,
        width,
        height,
      );
      this.maskCanvasCtx.save();
    }
  }

  private resetStatus() {
    Object.assign(this, {
      moving: false,
    });
  }

  private handleMouseDown(e: any): void {
    this.moving = true;
    const { offsetX, offsetY } = e;
    this.startY = offsetY + 1;
    this.startX = offsetX + 1;
    this.offsetTop = this.top;
    this.offsetLeft = this.left;
    this.saveMaskCanvasData();
  }

  private handleMouseMove(e: any): void {
    const { offsetX, offsetY, target } = e;
    const { clipWidth, clipHeight, top, left } = this;
    const path = this.getPath2DRect(left, top, clipWidth, clipHeight);
    this.setCursor(offsetX, offsetY, target);
    if (this.moving) {
      if (this.isPointInClip(offsetX, offsetY, path)) {
        this.moveClip(offsetX, offsetY);
      } else {
        // 判断是否在操作点上
        const { menuPoints } = this;
        const point = menuPoints.find((obj: {[key: string]: any}) => {
          const { x, y } = obj;
          const pointPath = this.getPath2DRect(x - 4, y - 4, 9, 9);
          return this.isPointInClip(offsetX, offsetY, pointPath);
        });
        if (point) {
          this.pointMove(point, offsetX, offsetY);
        }
      }
    }
  }

  private isBorderRight(left: number): boolean {
    const { clipWidth, width } = this;
    if (left + clipWidth >= width) {
      return true;
    }
    return false;
  }

  private isBorderLeft(left: number): boolean {
    if (left <= 0) {
      return true;
    }
    return false;
  }

  private isBorderTop(top: number): boolean {
    if (top <= 0) {
      return true;
    }
    return false;
  }

  private isBorderBottom(top: number): boolean {
    const { clipHeight, height } = this;
    if (top + clipHeight >= height) {
      return true;
    }
    return false;
  }

  private handleMouseUp(e: any): void {
    this.moving = false;
  }

  private moveClip(offsetX: number, offsetY: number) {
    this.resetCanvasData();
    const {
      startX,
      startY,
      offsetTop,
      offsetLeft,
      clipWidth,
      clipHeight,
      width,
      height,
    } = this;
    this.left = offsetX - (startX - offsetLeft) + 1;
    this.top = offsetY - (startY - offsetTop) + 1;

    const isBorderRight = this.isBorderRight(this.left);
    const isBorderLeft = this.isBorderLeft(this.left);
    const isBorderTop = this.isBorderTop(this.top);
    const isBorderBottom = this.isBorderBottom(this.top);

    if (isBorderRight) {
      this.left = width - clipWidth - 2;
    }
    if (isBorderLeft) {
      this.left = 2;
    }
    if (isBorderTop) {
      this.top = 2;
    }
    if (isBorderBottom) {
      this.top = height - clipHeight - 2;
    }
    this.setClipBorder();
  }
  private pointMove(obj: any, offsetX: number, offsetY: number) {
    const { position } = obj;
    console.log(position);
    const pointMoveMap: {[key: string]: any} = {
      topBottomRightMiddle: this.setClipWidth,
    };
    pointMoveMap[position].call(this, offsetX, offsetY);
    this.setClipBorder();
  }

  private setClipWidth(offsetX: number, offsetY: number) {
    const { startX, startY, offsetTop, offsetLeft } = this;
    const offset = offsetX - startX + 1;
    this.clipWidth = this.clipWidth + offset + 4;
  }
}
</script>
