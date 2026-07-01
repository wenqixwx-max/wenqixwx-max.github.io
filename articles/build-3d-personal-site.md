---
title: "用 Next.js + Three.js 在 GitHub Pages 上跑 3D 个人主页"
date: "2026-06-30T16:30:00+08:00"
excerpt: "技术干货：怎么在静态托管的环境里集成 3D 模型、动画、还要能发布文章。一份踩坑全记录。"
tags: ["技术", "Next.js", "Three.js", "GitHub Pages"]
---

# 在 GitHub Pages 上跑 3D 个人主页

最近折腾了一个新项目：把这只小精灵（首页转圈圈的那位）放到 GitHub Pages 上跑起来。
听起来简单，实际踩了不少坑，记录一下。

## 技术栈

- **Next.js 16** + TypeScript
- **Three.js** 通过 `@react-three/fiber` + `@react-three/drei`
- **Framer Motion** 做动画
- **Tailwind CSS** + shadcn/ui 做样式
- 部署到 **GitHub Pages**

## 难点 1：3D 模型加载

模型是 Blender 导出的 glTF 格式，包含：

```
wenqi.gltf    # 模型结构
wenqi.bin     # 顶点数据
Image_0.jpg   # 纹理 0
Image_1.jpg   # 纹理 1
Image_2.jpg   # 纹理 2
```

把整个文件夹放到 `/public/wenqi-model/` 下，用 `useGLTF` 加载即可：

```tsx
import { useGLTF } from "@react-three/drei";

function Model({ url }: { url: string }) {
  const { scene } = useGLTF(url);
  return <primitive object={scene} scale={1.15} />;
}
```

为了让它"活起来"，加了几个细节：

- 自动旋转 + 上下浮动
- 鼠标移动时相机微微跟随
- 加了 `Sparkles` 粒子让画面更有氛围
- 用 `ContactShadows` 让模型"接地"

## 难点 2：GitHub Pages 是纯静态的

GitHub Pages 不支持任何后端。文章发布怎么办？

我的方案是：

1. 文章以 Markdown 文件形式存在仓库里（`public/articles/*.md`）
2. 一个 prebuild 脚本扫描这些文件，生成 `index.json`
3. 前端 fetch 这个 JSON 拿到文章列表
4. 想发新文章时，通过 GitHub Contents API 把 Markdown 文件 commit 到仓库
5. GitHub Actions 自动重新构建并部署

这样既保留了"后台"的体验，又完全是静态的。✨

## 难点 3：性能优化

3D + 粒子 + 动画，很容易卡。我做了这些优化：

- 用 `dynamic(() => import(...), { ssr: false })` 让 3D 只在客户端加载
- 限制了 Canvas 的 `dpr` 在 `[1, 2]` 之间
- 粒子数量根据屏幕大小动态计算
- 用 `useMemo` 缓存克隆的模型场景

## 结语

最终效果你已经在首页看到了。如果你也想搭一个类似的网站，
欢迎 fork 我的仓库，把里面的内容换成你自己的就好。

祝你折腾愉快！🚀
