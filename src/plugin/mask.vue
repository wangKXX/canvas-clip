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
import { Point } from "./interface";

@Component
export default class ClipMask extends Vue {
  public $refs!: {
    mask: HTMLDivElement;
    maskCanvas: HTMLCanvasElement;
  };

  @Prop() public width!: number; // 图片初始宽度
  @Prop() public height!: number; // 图片初始高度
  @Prop() public initClipWidth!: number; // 裁剪区初始宽度
  @Prop() public initClipHeight!: number; // 裁剪区初始高度

  public left: number = 1;
  public top: number = 1;
  public cacheTop: number = 1;
  public cacheLeft: number = 1;
  public startX!: number;
  public startY!: number;
  public offsetLeft!: number;
  public offsetTop!: number;
  public startPointX!: number;
  public startPointY!: number;
  public moving: boolean = false;
  public clipHeight: number = 300;
  public clipWidth: number = 300;
  public cacheClipHeight: number = 300;
  public cacheClipWidth: number = 300;
  public timer!: any;
  public selectPoint!: any;
  public maskCanvasCtx!: any;
  public offset: number = 0;
  public maskCanvasData!: any;
  public borderPath!: any;
  public menuPoints!: Point[];
  public isPointMoving: boolean = false;
  public currentPoint!: any;

  @Watch("clipHeight")
  @Watch("clipWidth")
  @Watch("top")
  @Watch("left")
  public onWidthChange(value: any) {
    this.$emit("widthHeightChange", {
      width: this.clipWidth,
      height: this.clipHeight,
      top: this.top,
      left: this.left,
    });
  }

  @Watch("width")
  @Watch("height")
  public widthChange(value: any) {
    this.initMaskCanvas();
  }

  @Watch("initClipWidth")
  @Watch("initClipHeight")
  public oninitChange(value: any) {
    Object.assign(this, {
      clipHeight: this.initClipHeight || 300,
      clipWidth: this.initClipWidth || 300,
    });
    this.initMaskCanvas(true);
  }

  public isInMenuPart(offsetX: number, offsetY: number): boolean {
    const { clipWidth, clipHeight, top, left } = this;
    const path = this.getPath2DRect(left, top, clipWidth, clipHeight);
    const { menuPoints } = this;
    const point = menuPoints.find((obj: { [key: string]: any }) => {
      const { x, y } = obj;
      const pointPath = this.getPath2DRect(x - 4, y - 4, 9, 9);
      return this.isPointInClip(offsetX, offsetY, pointPath);
    });
    return Boolean(this.isPointInClip(offsetX, offsetY, path) || point);
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
    this.$emit("sendSelf", this);
  }

  private destroyed() {
    document.removeEventListener("mouseup", this.resetStatus);
  }

  private initMaskCanvas(setTopLeft?: boolean) {
    const { maskCanvas } = this.$refs;
    const { width, height } = this;
    maskCanvas.width = width;
    maskCanvas.height = height;
    this.maskCanvasCtx = maskCanvas.getContext("2d");
    this.maskCanvasClear();
    this.dashBorderLinemarch();
    if (!setTopLeft) {
      this.setInitTopLeft();
    }
  }

  private setInitTopLeft() {
    const { clipWidth, clipHeight, width, height } = this;
    this.left = (width - clipWidth) / 2;
    this.top = (height - clipHeight) / 2;
  }

  private maskCanvasClear() {
    const { width, height } = this;
    this.maskCanvasCtx.clearRect(0, 0, width, height);
    this.maskCanvasCtx.fillStyle = "rgba(0, 0, 0, .4)";
    this.maskCanvasCtx.fillRect(0, 0, width, height);
  }

  private setClipBorder(position?: string) {
    const { clipWidth, clipHeight, top, left, maskCanvasCtx, offset } = this;
    this.maskCanvasClear();
    maskCanvasCtx.beginPath();
    maskCanvasCtx.setLineDash([4, 2]);
    maskCanvasCtx.lineDashOffset = -offset;
    maskCanvasCtx.lineWidth = 1;
    maskCanvasCtx.strokeStyle = "#fff";
    maskCanvasCtx.rect(left, top, clipWidth, clipHeight);
    maskCanvasCtx.closePath();
    maskCanvasCtx.stroke();
    maskCanvasCtx.restore();
    this.borderPath = maskCanvasCtx;
    this.setClipMask();
    this.setMenuPoints(position);
  }

  private setDoMenuPoint(x: number, y: number, color: string) {
    const { maskCanvasCtx } = this;
    maskCanvasCtx.beginPath();
    maskCanvasCtx.fillStyle = color;
    maskCanvasCtx.rect(x, y, 9, 9);
    maskCanvasCtx.closePath();
    maskCanvasCtx.fill();
    maskCanvasCtx.restore();
  }

  private setMenuPoints(position?: string) {
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
    this.menuPoints.forEach((item: Point) =>
        this.setDoMenuPoint(item.x - 4, item.y - 4, item.position ===  position ? "#36f" : "#fff"),
    );
  }

  private setCursor(x: number, y: number, target: HTMLCanvasElement): void {
    const { clipWidth, clipHeight, top, left } = this;
    const path = this.getPath2DRect(left, top, clipWidth, clipHeight);
    const { menuPoints } = this;
    const point = menuPoints.find((obj: { [key: string]: any }) => {
      const { x: X, y: Y } = obj;
      const pointPath = this.getPath2DRect(X - 4, Y - 4, 9, 9);
      return this.isPointInClip(x, y, pointPath);
    });
    if (this.isPointInClip(x, y, path) && !point) {
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

  /**
   * 鼠标是否在裁剪区域
   */
  private isPointInClip(x: number, y: number, path: Path2D): boolean {
    const { maskCanvasCtx, top, left, clipWidth, clipHeight } = this;
    return maskCanvasCtx.isPointInPath(path, x, y);
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
    this.cacheClipWidth = this.clipWidth;
    this.cacheClipHeight = this.clipHeight;
    this.cacheTop = this.top;
    this.cacheLeft = this.left;
    this.saveMaskCanvasData();
  }

  private handleMouseMove(e: any): void {
    const { offsetX, offsetY, target } = e;
    const { clipWidth, clipHeight, top, left } = this;
    const path = this.getPath2DRect(left, top, clipWidth, clipHeight);
    const { menuPoints } = this;
    this.setCursor(offsetX, offsetY, target);
    if (this.moving) {
      let point: any = menuPoints.find((obj: { [key: string]: any }) => {
        const { x, y } = obj;
        const pointPath = this.getPath2DRect(x - 4, y - 4, 9, 9);
        return this.isPointInClip(offsetX, offsetY, pointPath);
      });
      if (
        this.isPointInClip(offsetX, offsetY, path) &&
        !this.isPointMoving &&
        !point
      ) {
        this.moveClip(offsetX, offsetY);
      } else {
        // 判断是否在操作点上
        this.isPointMoving = true;
        point = this.currentPoint || point;
        if (point) {
          this.currentPoint = point;
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
    this.isPointMoving = false;
    this.currentPoint = null;
    this.cacheClipWidth = this.clipWidth;
    this.cacheClipHeight = this.clipHeight;
    this.cacheTop = this.top;
    this.cacheLeft = this.left;
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
    const pointMoveMap: { [key: string]: any } = {
      topBottomRightMiddle: this.setClipWidth,
      bottomLeftMiddle: this.setClipHeight,
      bottomRight: this.setBottomRight,
      topBottomLeftMiddle: this.setClipWidthLeft,
      topLeftMiddle: this.setClipHeightTop,
      topLeft: this.setClipTopLeft,
      topRight: this.setClipTopRight,
      bottomLeft: this.setClipBottomLeft,
    };
    pointMoveMap[position].call(this, offsetX, offsetY);
    this.setClipBorder(position);
  }

  private setClipWidth(offsetX: number, offsetY: number) {
    const { startX, startY, offsetTop, offsetLeft, cacheClipWidth } = this;
    const offset = offsetX - startX;
    this.clipWidth = cacheClipWidth + offset;
  }

  private setClipHeight(offsetX: number, offsetY: number) {
    const { startX, startY, offsetTop, offsetLeft, cacheClipHeight } = this;
    const offset = offsetY - startY;
    this.clipHeight = cacheClipHeight + offset;
  }

  private setBottomRight(offsetX: number, offsetY: number) {
    this.setClipWidth(offsetX, offsetY);
    this.setClipHeight(offsetX, offsetY);
  }

  private setClipWidthLeft(offsetX: number, offsetY: number) {
    const {
      startX,
      startY,
      offsetTop,
      offsetLeft,
      cacheClipWidth,
      cacheLeft,
    } = this;
    const offset = offsetX - startX;
    this.left = cacheLeft + offset;
    this.clipWidth = cacheClipWidth - offset;
  }

  private setClipHeightTop(offsetX: number, offsetY: number) {
    const {
      startX,
      startY,
      offsetTop,
      offsetLeft,
      cacheClipHeight,
      cacheTop,
    } = this;
    const offset = offsetY - startY;
    this.clipHeight = cacheClipHeight - offset;
    this.top = cacheTop + offset;
  }

  private setClipTopLeft(offsetX: number, offsetY: number) {
    this.setClipWidthLeft(offsetX, offsetY);
    this.setClipHeightTop(offsetX, offsetY);
  }

  private setClipTopRight(offsetX: number, offsetY: number) {
    this.setClipWidth(offsetX, offsetY);
    this.setClipHeightTop(offsetX, offsetY);
  }

  private setClipBottomLeft(offsetX: number, offsetY: number) {
    this.setClipHeight(offsetX, offsetY);
    this.setClipWidthLeft(offsetX, offsetY);
  }
}
</script>
