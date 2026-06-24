# Taro 多端构建与目录设计

## 构建命令

项目通过 `ena` 包装 Taro 构建，命令按端和环境拆分：

```json
"local": "ena dev --type=h5 --ena_env=prod",
"local:weapp": "ena dev --type=weapp --ena_env=prod",
"build": "ena build --type=h5 --ena_env=prod",
"build:weapp": "ena build --type=weapp --ena_env=prod"
```

可复用经验：多端项目必须把“端”和“环境”两个维度拆清楚。端决定平台能力，环境决定接口域名、资源前缀、监控环境和凭证。

## Taro 配置核心点

`config/index.ts` 里有几个关键配置：

- `framework: 'react'`
- `sourceRoot: 'src'`
- `outputRoot: 'dist'`
- `designWidth()` 返回 `390`
- 关闭小程序端默认 `pxtransform`
- 开启 `@tarojs/plugin-http`
- 注入 `process.env.*` 环境变量
- 用 `TsconfigPathsPlugin` 支持 `@/*` 路径别名
- H5 开启 browser 路由和增强动画
- 小程序开启 `optimizeMainPackage`

这些配置决定了项目的基础工程形态。

## 页面与分包

`src/app.config.ts` 中：

- 主包只有 `pages/town/index`。
- 用户、webview、carrier、徽章、名片设置都拆到分包。
- 对关键分包配置 `preloadRule`。

经验：

- 小程序主包尽量小，只放首屏必须页面。
- 次级页面进入分包，降低首次下载压力。
- 高频跳转或首屏后马上可能访问的分包，可以配置预加载。
- H5 没有小程序分包的同等收益，所以分包设计主要服务微信端。

## 平台文件拆分

项目中有典型平台文件：

- `src/components/CanvasTemplate/index.tsx`
- `src/components/CanvasTemplate/index.h5.tsx`
- `src/components/QRCode/index.tsx`
- `src/components/QRCode/index.h5.tsx`
- `src/components/SharePopup/index.tsx`
- `src/components/SharePopup/index.h5.tsx`
- `src/hooks/useUpload/index.ts`
- `src/hooks/useUpload/index.weapp.ts`
- `src/utils/audioManager.ts`
- `src/utils/audioManager.h5.ts`
- `src/utils/tlog.ts`
- `src/utils/tlog.h5.ts`

规则：

- 行为差异小：用 `process.env.TARO_ENV` 分支。
- 能力模型差异大：用 `.h5.ts/.weapp.ts` 拆文件。
- 组件渲染完全不同：用平台组件入口拆文件。

## 从 0 到 1 的目录建议

```text
src/
├── app.config.ts
├── app.tsx
├── app.scss
├── pages/
│   └── home/
│       ├── index.tsx
│       ├── index.scss
│       └── index.config.ts
├── components/
├── model/
├── store/
├── services/
├── hooks/
├── utils/
├── constants/
└── types/
```

建议每个页面都有自己的 `index.config.ts`，平台差异配置可以补 `index.config.h5.ts` 或 `index.config.weapp.ts`。

## 常见坑

- 不要默认 H5 和小程序生命周期完全一致。
- 不要在业务代码里散落环境变量读取，应先收敛到 constants 或 env utils。
- 不要把小程序专属 API 直接写进通用组件。
- 不要把 Taro 默认 `pxTransform` 和自定义单位转换混用。
- H5 publicPath 和资源前缀必须独立考虑，否则线上资源容易 404。

## 面试表达

> 我会把多端构建拆成端、环境、页面包和平台文件四层。端通过 `--type=h5/weapp` 控制，环境通过构建变量注入；页面层小程序主包只保留首屏，其他页面走分包和预加载；平台差异小的用条件分支，差异大的用 `.h5.ts/.weapp.ts`。这样既能复用业务逻辑，又不会让平台补丁污染通用代码。

## 代码参考

- `package.json`
- `config/index.ts`
- `src/app.config.ts`
- `src/components/CanvasTemplate`
- `src/hooks/useUpload`
