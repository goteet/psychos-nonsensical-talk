---
layout: post
title: Unity SRP 概念介绍
---

# Unity Scriptable Render Pipeline 概念介绍

*Future of Rendering in Unity*

2020-05-02 快速翻一遍

这是一个很早期介绍概念的 Slider 的翻译。

本来是希望找到和 Frame Graph 对应的概念，通过使用方式或介绍推导一下具体设计方法的。似乎概念还不太一样，应该去看看文中提到的 Tobias Pesson 在 blog 里写的内容。

几个比较有帮助的链接如下：

[Frame graphs · gfx-rs/gfx Wiki](https://github.com/gfx-rs/gfx/wiki/Frame-graphs) 有用的讨论设计的链接汇总。

[Unity Custom Render Pipeline Tutorial](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/)，Catlike的教程。

[Unity Scriptable Render Pipeline manual](https://docs.unity3d.com/Manual/ScriptableRenderPipeline.html)， 官方介绍手册。

好吧，我们开始吧。

## 问题简述

不同游戏类型的画面表现效果。有不一样的侧重点：

- 2D 平台游戏、手机文字解密游戏
- HD高质量画质游戏；风格化渲染游戏等。
- 大量UI、文字；真实画面效果 vs. Toon Shading光照。

现阶段，Unity已经能支持的管线能力如下（理论上）：

- 不同的管线：Forward Shading、Deferred Shading
- 可配置：自定义光照、材质；Compute shaders、自定义后处理效果，Command Buffers等。
- 跨平台适应性良好。

实际上，Unity 这些方面支持程度很差：

- 对于开发者来说，就像是一个巨大的黑盒。
- 如果不了解内部设计，很难正确配置。
- 也不容易拓展，渲染性能一般。
- 而且似乎在往大而全一刀切（One Size Fits All） 的方向去发展。
- 并没有针对不同的平台的有点去做特性适配
- 还不容易修改当前的行为（Behavior）。

## 解决思路

### 目标是什么？

- 很少的C++ 代码
- 暴露相关的APIs。
- 能在c#里写高层的渲染循环逻辑。

我们希望渲染器是什么样子的呢？

- 精益：最小的 surface area；可测试；松耦合。
- 用户为本：
  - 能够作为扩展，或者直接在用户的项目里
  - 可以调试
  - 可拓展和修改
  - 容易修改：快速的修改迭代时间。
- 优化：快；为特殊的平台和做优化；可以在项目不需要时去除某些部分。
- 显式的：只做用户需要的；NO Magic；干净的API。

为了解决这个问题，提出了Script Render Pipeline 的概念。

- 如果性能关键区域，会在引擎/C++处完成。如果以后c#能被优化，也会考虑移出来。
- 引擎c++代码：裁剪；排序；Batching；渲染物体集合；渲染平台抽象。
- c# or shader：摄像机设置；光照、阴影设置；每一帧的 Render Passes 设置和逻辑；compute/shader 代码

#### 前人的经验

除了我们，别人也有这么做过的：

- “[Benefits of a data-driven renderer](https://www.slideshare.net/tobias_persson/bstech-gdc2011)”, Tobias Persson, GDC 2011
- “[Destiny’s Multi-Threaded Rendering Architecture](http://advances.realtimerendering.com/destiny/gdc_2015/)”, Natalya Tatarchuk, GDC 2015
- “[Framegraph: Extensible Rendering Architecture in Frostbite](https://www.ea.com/frostbite/news/framegraph-extensible-rendering-architecture-in-frostbite)”, Yuriy O’Donell, GDC



这三个论文，第一个不太好找：所以我直接把 PDF 在这里翻译一下渲染实现的部分：

用 Json 来描述整个渲染管线（可以热加载），如下图所示：

![render_config_overview]({{ site.baseurl }}/images/2020-05-02-unity-srp-preivew/render_config_overview.png)

图：完整的渲染管线声明结构。

**全局资源**：最一开始的时候，所有的 Render Target （资源，图上都是rt）都要明确声明好，通过名字设置依赖关系，启动的时候就会被进行分配。

**Layer配置**：定义所有可见 Batch 的渲染顺序。每一层的顺序是在声明的时候就指定好的，通过 Shader System 来确定哪一层被渲染。（有可能会多定义一些Debug Layer？）

```json
"default":{
  { "name":"depth_prepass", "ds_target":"ds_buffer", "sort":"F2B" },
  {
    "name":"gubffer", "sort":"F2B",
    "rt_targets":"albedo normal",
    "ds_target":"ds_buffer",
    "profiling_scope":"gbuffer"
  },
  {
    "name":"deferred_shading", "res_gen":"deferred_shading",
    "rt_targets":"light_accumulation",
    "ds_target":"ds_buffer"
  },
  { "name":"skydome", ...},
  { "name":"reflection", ...},
  { "name":"fog_apply", ...},
  { "name":"semi_transparency", ...},
  { "name":"reansparency", ...},
  { "name":"tone_mapping", ... },
  {
    "name":"post_process", "res_gen":"post_processing",
    "rt_target":"back_buffer",
    "ds_target":"ds_buffer",
    "profiling_scope":"post_processing"
  }
}
```



- 每一层都是一个粗粒度的阶段，类似 Unity Pipeline 里的 LightMode。
- 会通过名字指定好渲染输出的RT；排序算法。

- 可选的资源生成器：复用资源；类似后处理的只是做Graph.Blit操作。

**Viewport**：最后是通过Viewport整合所有的渲染操作的。并通过 View 来确认哪个配置要被使用，有可能不同的View有不同的 configuration。

还有一部分本地资源集，比如说其他配置里没有的 current adapted luminance 之类的图。

最后GamePlay程序员可以直接调用来渲染世界。

```c++
render_world(pWorld, pCamera, Viewport);
```

<font color="red">值得一提的是，Tobias Persson在最新的blog里全面转向了 Frame Graph，抛弃了DOD设计，看 frame graph 翻译里的链接。</font>

#### Unity所做的技术选型

这里有两种技术选型，是配置驱动的（DOD），还是代码驱动的？

代码：程序员喜欢代码，相对于面条图来说；有些情况下需要依赖游戏的逻辑和状态进行分支切换。

### 主要的 c# APIs

裁剪特定的Views，渲染可见对象的子集：

- 通过使用的材质信息；
- 排序标志；
- 或者特定features
  （with "What kind of per-object data to setup", Light Probe, per-objet-light lists, etc.）

现在已经有的 API是：

- 设置 Render Passes / Reder Targets
- 设置 Shader Constants / Global Resources
- 派发 Computer Shaders
- 绘制单独的 Mesh： Special FX / Post FX

- API 通过构造一个 Command Buffer 获得稍后分析和执行的能力。（Retain Mode）

#### 性能考量

只是高层的帧结构定义，没有的单个可见物体的操作。

关键的是比老的 C++ 还要快点（……）

而且正在研究一些关于c#多线程/no-GC 的东西。

![render_loop_comparision]({{ site.baseurl}}/images/2020-05-02-unity-srp-preivew/render_loop_comparision.png)

图：新旧代码比较。（如果老代码也是很用心的去写的话，才有意义，否则新管线用C++还会更快？）

#### 通过SRP支持不同的管线

尝试把引擎内的管线挪出来

- 用于PC、主机、高端移动设备的 HDRP
- 用于低端移动设备的 LWRP。
- 用于VR的管线。

##### HD Pipeline（HDRP?)

支持 PBR、GX、区域光照、FPTL/Clustered，aniso GGX，Layered，SSS（反正一堆流行技术）。

支持 Compute Shader 。

链接失效了，估计现在改成HDRP了吧。之前说是在5.6版本可以支持。