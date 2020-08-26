# wang-image-clip

## Using npm:
```
npm i wang-image-clip --save
```
## params:

参数名|描述|类型|默认值
--|:--:|:--:|--:
src|传入的图片地址|image|--156 5373 8086
height|图片的高度(宽度将进行等比缩放)|number|500
clipHeight|裁剪区域初始高度|number|200
clipWidth|裁剪区域初始宽度|number|200

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