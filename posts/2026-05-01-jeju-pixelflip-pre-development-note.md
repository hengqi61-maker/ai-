# 济州岛 PixelFlip Pre：一次视觉演示原型的开发记录

日期：2026-05-01

这次做 `Jeju PixelFlip Pre`，核心目标不是再做一个普通旅游网页，也不是做一套线性 PPT，而是探索一种“可点击的视觉地图式演示”：观众看到的是一张像素风济州岛微缩世界，演讲者通过点击岛上的地形、路线、景点和文化区域，把讲述推进到不同的准备好页面。

## 起点：把 PPT 变成一张可探索的知识地图

传统 PPT 的问题是线性、文字先行、页面之间容易断裂。这个原型尝试反过来：先让背景图承担语境，让观众先读图，再由演讲者补充解释。

因此项目里每个页面都不是孤立幻灯片，而是一个场景节点：

- `cover-overview` 作为总览入口
- `spatial-structure` 解释济州岛的空间结构
- `core-sight-map` 承担主要景点分支
- `recommended-routes` 和 `travel-advice-summary` 负责路线与建议收束
- Hallasan、Seongsan、Udo、Jusangjeolli、Waterfalls、Food、Dongmun Market 等扩展页作为可点击深挖章节

最后形成了 17 个 draft 场景：每个都有 scene graph、内容卡片、背景资产和热点关系。这个阶段的重点不是无限扩展页面，而是让一个可演示的核心循环稳定下来。

## 中间最大的判断：不做实时生成，先做 prepared visual browser

一开始很容易被“动态画布”“实时生成”“活的地图”吸引，但这次项目最终回到 prepared visual browser 的路线。

原因很实际：

- 演示场景需要稳定，不应该依赖现场生成结果
- 视觉连续性比技术炫技更重要
- 文字层要能维护，不能全部烘焙进图片
- 观众感受到的是“进入一个准备好的世界”，而不是看系统现场拼装

所以每个场景采用提前设计、提前审查、提前集成的方式。真正重要的是点击之后是否像进入同一个世界的局部，而不是页面数量有多少。

## 视觉验收：能演示，但还不能标记 reviewed

用 Computer Use + Chrome 对当前原型做了一轮视觉验收后，结论比较明确：

- 17 个场景的核心背景都能加载
- `ext-seongsan`、`ext-jusangjeolli`、`ext-waterfalls`、`ext-udo`、`ext-dongmun-market` 的视觉完成度较强
- `recommended-routes` 和 `travel-advice-summary` 的路线/季节/建议表达比较稳定
- Presenter View 可以用，但仍有内部工具感
- QA 模式下 missing-assets 条会遮挡画面，只适合验收，不适合对外展示截图

这说明项目已经能进入 demo rehearsal，但不能直接叫 reviewed。原因不是“图不好看”，而是交互状态还没有完全稳。

## 最关键的 bug：URL 回来了，画面没回来

验收中复现了一个正式演示前必须修的问题：

从 `core-sight-map` 点击进入 `ext-seongsan` 后，再点击 Back，地址栏已经变成：

```text
?scene=core-sight-map&qa=1
```

但画面和 DOM 仍然停留在 `ext-seongsan`。

这不是简单的“页面没刷新”，根因是状态模型分成了两层：

- URL query 里有 `scene`
- 应用内部 Zustand store 里也有 `currentSceneId`

原实现只在初始化时读取一次 URL，之后 scene 渲染完全依赖 Zustand store。导航提交时又用 `replaceState` 写 URL，没有建立浏览器 Back/Forward 与 store 之间的反向同步。结果就是浏览器历史变了，地址栏变了，但 React 订阅的内部 scene state 没变，画面自然不会重渲染。

这个问题暴露了一个很典型的原型风险：视觉上“能点”不等于状态机可靠。对演示工具来说，URL、store、DOM、画面标题必须一致，否则现场讲解很容易翻车。

## 修复方向：最小双向同步，而不是重构导航系统

这次没有重构整个架构，而是做最小稳定性修复：

- URL helper 支持 `pushState` / `replaceState`，并保留 `qa=1` 等 query 参数
- 热点、Next、Previous、Overview 等正常导航使用 `pushState`
- 应用内 Back 使用已有 history stack，并用 `replaceState` 对齐 URL
- 增加从 URL 应用 scene 的入口，专门处理浏览器 `popstate`
- `popstate` 触发时清理 pending transition，避免旧 scene 被锁住
- 对浏览器 Forward 进入 extension 的情况恢复父级 Back 栈

修完后验证路径是：

1. 打开 `?scene=core-sight-map&qa=1`
2. 点击 `Seongsan Ilchulbong`
3. 确认 URL、DOM 标题、画面标题都进入 `ext-seongsan`
4. 浏览器 Back 回到 `core-sight-map`
5. 浏览器 Forward 回到 `ext-seongsan`
6. 应用内 Back 也能回到父页，并保留 `qa=1`

自动检查也补上了 URL helper 和 navigation store 的测试，避免后续改动再次破坏这个链路。

## 这次开发带来的几个经验

第一，演示型产品的状态一致性比页面数量重要。17 个场景已经足够证明体验方向，继续扩展新场景不如先把 Back/Forward、转场和 QA 流程修稳。

第二，prepared visual browser 是一个适合短期原型的路线。它没有实时生成那么炫，但更适合演示、复盘和逐步打磨。

第三，视觉连续性要有验收标准。比如从 `core-sight-map` 到 `ext-seongsan`，子场景必须像从父图里的地标 zoom in 出来，而不是突然跳到一张漂亮但无关的图。

第四，QA 模式和观众模式必须分清。热点框、missing-assets、debug 文案对开发很有用，但不能默认暴露给观众。

## 当前状态

现在这个项目更准确的状态是：

```text
draft / demo rehearsal ready candidate
```

还不能提升到 reviewed。剩余阻塞包括：

- 17 个场景还需要完整 Browser QA
- 父子场景 anchor-continuity 还需要复验
- `core-sight-map` 的热点密度仍需校准
- 右上 breadcrumb 和折叠内容 pill 在长标题页面仍需继续打磨
- Presenter View 还需要从内部工具态升级到演示可读态
- foreground、ambient、thumbnail 仍是可选缺失层

但这次最大的收获是，原型已经从“看起来能跑”推进到了“关键演示路径开始可验证”。下一步应该继续压实演示链路，而不是急着加更多页面。