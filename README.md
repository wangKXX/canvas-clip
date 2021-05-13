# wang-image-clip

## Using npm:
```
npm i wang-image-clip --save
```
## params:

参数名|描述|类型|默认值
--|:--:|:--:|--:
src|传入的图片地址|image|--
height|图片的高度(宽度将进行等比缩放)|number|500
clipHeight|裁剪区域初始高度|number|200
clipWidth|裁剪区域初始宽度|number|200
showPreview|是否显示预览区域|boolen|true
showControls|是否显示操作栏|boolen|true

## event

事件名|返回值
--|--:
clipCallBack|裁剪后的图片

## $ref api

clipImage: 手动触发裁剪，可用于用户自定义何时触发裁剪事件

## attention
仅支持同域和设置了跨域的图片地址
## example:
```
<template>
  <div id="app">
    <clip :src="require('./assets/logo.png')" />
  </div>
</template>

<script>
import clip from "wang-image-clip";
import "wang-image-clip/dist/wang-image-clip.css";

export default {
  name: 'app',
  components: {
    clip
  }
}
</script>
```