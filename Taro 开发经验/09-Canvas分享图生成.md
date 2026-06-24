# Canvas 分享图生成

## 项目场景

Papertown 有小镇地图分享和海报预览能力。这个能力跨端差异很大：

- 微信小程序端：使用 `wxml2canvas-2d` 原生组件。
- H5 端：使用 `html2canvas-pro` 截 DOM。

项目通过 `src/components/CanvasTemplate/index.tsx` 和 `index.h5.tsx` 拆分实现，向外暴露统一的 `generateImg()`。

## 小程序端方案

小程序端：

- 构建时把 `wxml2canvas-2d/miniprogram_dist` copy 到 `dist/components/wxml2canvas-2d/`。
- `CanvasTemplate` 渲染原生 `<wxml2canvas-2d>`。
- 通过 `page.selectComponent(#id)` 获取组件。
- 调用 `draw()` 和 `toTempFilePath()` 导出图片。
- 失败时把 scale 降级到 `1 / pixelRatio` 重绘。

经验：小程序 Canvas 导出失败时，不要只 toast。可以做降级重绘，尤其是高 DPR 设备。

## H5 端方案

H5 端：

- 不渲染 Canvas 组件。
- 通过 `Taro.createSelectorQuery()` 找 DOM 容器。
- 用 `html2canvas-pro` 截图。
- 宽高传 CSS 尺寸，避免重复乘 DPR。
- 在 `onclone` 中修补克隆 DOM。

H5 特殊处理：

- CSS `background-image` 替换为真实 `<img>`，提升清晰度。
- 精灵图动画先截首帧，再替换进克隆 DOM。
- 等替换图片加载完成再截图。
- 支持忽略指定 class 的元素。

## 海报预渲染与揭示时机

`usePosterGeneration` 里有一个很好的体验优化：

- 同时生成“带用户信息”和“不带用户信息”两版海报。
- 同一 variant 并发调用复用 in-flight promise。
- 两版都生成成功后才 reveal 预览。
- 切换是否展示用户信息时，只切换缓存图的 visibility，不重新生成。

这样避免了：

- 首图显示后第二张追加导致闪烁。
- 用户切换开关时白屏。
- 重复 draw 同一个 canvas。

## Canvas 资源规则

小程序 iOS Canvas 不支持 SVG 图片。经验：

- 参与 Canvas 截图的资源使用 PNG/JPG。
- 动态颜色不要客户端生成 SVG，可以准备多份 PNG。
- 背景图、精灵图、头像、边框都要考虑 CORS、加载完成和输出清晰度。
- 截图源容器的 CSS 尺寸决定输出分辨率，不要为了预览缩小源容器。

## 从 0 到 1 实现步骤

1. 设计统一接口：`generateImg(options): Promise<string>`。
2. 小程序端接入 `wxml2canvas-2d` 或原生 Canvas。
3. H5 端接入 `html2canvas-pro`。
4. 平台文件拆分。
5. 截图源和预览展示分离。
6. 资源加载完成后再截图。
7. 做失败兜底和 loading。
8. 做缓存，避免重复生成。

## 面试表达

> 分享图是项目里平台差异最大的能力之一。我们对外暴露统一 `generateImg`，小程序端用 `wxml2canvas-2d` 原生组件，H5 端用 `html2canvas-pro` 截 DOM。H5 截图时会在 clone DOM 中把 background-image 替换成 img，并提取精灵图首帧保证清晰度。分享弹窗会同时预生成两版海报，全部完成后再展示，切换时直接用缓存图，避免闪烁和重复生成。

## 代码参考

- `src/components/CanvasTemplate/index.tsx`
- `src/components/CanvasTemplate/index.h5.tsx`
- `src/pages/town/hooks/usePosterGeneration.ts`
- `src/pages/town/components/TownSharePopup/TownSharePopupBase.tsx`
- `config/index.ts`
