---
layout: post
title: frame-graph-extensible-rendering-architecture-in-frostbite
---
# Frame Graph - 寒霜引擎可拓展的渲染结构

2020.05.01 这回争取尽量少改，减少写作时间。



又被Graph调戏了。**Frame Graph** 是一种用来处理复杂渲染管线的设计模式。现在正被应用于工业界，使用它的动机，主要是为了隐藏在后台控制的Barriers处理，队列同步，内存对齐等操作，从而能够从更高的层次来抽象一帧的渲染管线。

这是一篇 GDC talk 的读书笔记。这篇演讲分为三个部分

- 十年的引擎发展带来的问题
- 介绍什么是Frame Graph
- 临时资源系统的概念

并指出使用 Frame Graph 能够给引擎带来的好处为：

- 提高了引擎的拓展性
- 简化了异步计算的复杂度
- 自动对齐 ESRAM
- 而且能节省**大量**的显存

## 简介

演讲在最开始给我们展示了寒霜引擎十年来的发展。

在这10年的发展中，引擎从最初的战地引擎，逐步变为一款能支持RPG、Racing、Sports、Action等不同游戏类型的通用引擎。现在已经在15款游戏中使用了。

其中，这十年里渲染系统在维持基本形态不变的情况下，增加了更多的系统，更复杂的耦合，变得更为庞大。对比演示如下图所示：

![rendering_architecture_comparision]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/rendering_architecture_comparision.png)

图：2007-2017 渲染系统的变化的对比。2017的结构并不完整，只是系统变得更复杂的演示。

随着集成大量的特性、变大的系统间的产生的更多通信，渲染系统不断扩张，让维护变得困难。

### World Renderer

本次演讲主要聚焦在 World Renderer 和其中的渲染特性上。

World Renderer 是一个代码驱动的架构，负责协调整合系统里所有的渲染工作，这些工作主要有：

- （通过Shading System）维护世界里所有的几何体。
- （通过Render Context）维护光照、后处理过程。
- 知道所有Views 和 Render Passes 的存在。

当然 World Renderer 也负责统筹在不同的系统中的设置，和渲染资源（RTs，Buffers），并负责分配他们。

![world_renderer_and_features]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/world_renderer_and_features.png)

图：Simplified Architecture 和 Render Passes。这是几年前战地4的Render Passes，现在加入了PBR之后更复杂了。

在2016之前的发展过程中，老的 World Renderer 存在如下问题：

- 显式的立即模式渲染。
- 显式的资源管理：手工定制的ESRAM管理，不同的项目组维护了不同的实现代码。
- 和渲染系统有非常紧密的耦合。
- 无法很好的拓展，项目组需要 Fork 出来进行定制。
- 代码膨胀，一共有15k行代码，并且单个函数超过2000行。维护，拓展和集成的代价很高。

### 模块化的 World Renderer 目标

为了解决上述问题，模块化的 World Renderer 被提出，并且需要满足以下目标：

- （拥有、描述）一帧完整的高层次信息。
- 具备更高的可拓展性：
  - 解耦、组件化各个代码模块；
  - 能自动进行资源管理
- 有更好的可视化和诊断工具。

为此，寒霜引擎在2016年重构了 World Renderer，并在渲染结构中加入了两个新的组件：

- Frame Graph：用来进行高层次的描述 Render Passes 和资源。
- 临时资源系统：资源分配和内存对齐。

![add_frame_graph_component]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/add_frame_graph_component.png)

图：新加入的组件在架构中位置。



## Frame Graph

### 整体印象

#### Frame Graph的设计理念

Frame Graph 是用来呈现每一帧都做了什么的一个组件。

- 构建每一帧的全局高层信息，包括
  - 简化的资源管理；
  - 简化的渲染管线配置；
  - 简化的异步执行和资源 Barriers。
- 允许自包含的高效渲染模块。
- 用来展示、Debug复杂的关键结构。

Frame Graph 是一个DAG，如下图所示：

![frame_graph_example]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/frame_graph_example.png)

图：用来实现 Deferred Shading 管线的Frame Graph示例。红线代表写数据，绿线代表读数据。

#### 数据呈现方式

虽然结构很简单，但是完整构造出来的图是很复杂的，所以他们用 GraphViz 输出了一个可以查找定位的 PDF。但由于图太大了，更多情况下，他们使用的是 Javascript + Json 的数据展示器，用于追踪在运行期每个Pass 内，数据构造、读取、写入和生命周期的信息。

视频太大就不放了，放个生成的节点的示例吧。

![graph_visualizer]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/graph_visualizer.png)

图：PDF内一些节点的示例。（用于反推实现……？）

#### Frame Graph的设计

设计总结Frame Graph 从立即模式的渲染转向了保留模式的渲染（Retained）。保留模式指的是声明性API，Frame Graph 通过API的声明，构造并维护渲染结构的层级结构，并在执行阶段进行更新。

- 渲染代码被划分到了不同的 Passes 里。
- 仍然保留了用C++代码来驱动架构，而非代码驱动。因为不想让大家花费巨大的努力去定义完整的管线结构，然后再关掉某些部分。

- 保留模式的渲染API分为三个阶段：Setup、Compile、Execute。

接下来开始分三个阶段讲解 Frame Graph 的设计。

### Frame Graph Setup 阶段

由于渲染配置会根据玩家的行动、或者debug的情况（后文有讲）经常动态的改变，所以寒霜引擎在每一帧都要重新构建Frame Graph。

为此，必须要保证Setup阶段就是要求快，就像（?as）在处理很小数量的 Render Passes 和资源。

在构建的过程中，需要完成二个任务：

- 定义所有的 Rendering/Compute Passes
- 为每一个Pass定义输入输出资源。

Setup流程只会构建一帧里需要的的渲染操作。构建流程和立即模式非常相似，但是不会产生实际的渲染指令，资源也都是虚拟的。整个构建过程中通过虚拟的资源句柄来输入输出。

#### 资源声明

每一个Pass都要明确定义好所有使用到的资源，包括他们的创建和读写操作。有一些 Pass 会有隐藏的Side-Efect，比如说从GPU回读数据。这些操作也都会要被显示标记出来。

有一些固定的RT，比如TAA使用的History Buffer，SSR使用的Back Buffer，可以被Import 到 Frame Graph 里。往这些Import资源写入的行为，应该被算作是Pass 的Side-Effect，否则会被后续的编译过程剔除（去除不会产生影响的Passes）。

下面给出几个 Render Pass 访问资源的示例。

##### 创建临时 RenderTarget 的例子

```c++
RenderPass::RenderPass(FrameGraphBuilder& builder)
{
	// Declare new transient resource
	FrameGraphTextureDesc desc;
	desc.width = 1280;
	desc.height = 720;
	desc.format = RenderFormat_D32_FLOAT;
	desc.initialSate = FrameGraphTextureDesc::Clear;
	m_renderTarget = builder.createTexture(desc);
}
```

![render_pass_create_rt]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/render_pass_create_rt.png)

图：RenderPass输出一个RT，和创建一张贴图差不多。

##### 读取一张贴图，写入另一张

```c++
RenderPass::RenderPass(FrameGraphBuilder& builder,
	FrameGraphResource input,
	FrameGraphMutableResource renderTarget)
{
	// Declare resource dependencies
	m_input = builder.read(input, readFlags);
	m_renderTarget = builder.write(renderTarget, writeFlags);
}

```

![render_pass_rw_rt]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/render_pass_rw_rt.png)

图：从输入读取RT，并输出到新的RT上。

Pass 写入RT的时候，会产生一个新的重命名句柄。重命名的目的是为了在 debug 资源修改时的一些顺序错误，比如同一张贴图被不同的Pass写入了。

重命名资源会强制指定 passes 之间的执行顺序。

#### 其他 Frame Graph 的骚操作

- **延迟创建资源**：当资源第一次被使用的时候才创建，并且根据Pass的使用情况自动设置Bind Flags。
  比较典型的就是Depth Buffer，虽然我们知道，进行3D渲染时需要一个，但是我们没必要关心我们的管线是否在使用Z Pre-Pass。Z Pre-Pass，G-Buffer，和前向管线都只是简单的写入到被提前定义好的 Depth Buffer 里。
- **推导资源的参数**：根据输入RT的尺寸和格式，构造相同参数的输出RT。根据使用情况，推导Bind Flags。
  每个句柄都附加了描述性的metadata。比如说Down Sample Pass，可以输出根据输入的信息，构建一个只有一半大小的输出资源。
- **MoveSubresource**：为资源创建Aliasing，和后续的Pass共享同一个资源数据。

#### 用反射做举例说明 MoveSubresource

一个通用的渲染管线可以被实现为输出一张简单的2D贴图资源，对应下图中 Deferred Shading Module 中输出一张 Lighting Buffer。

Lighting Buffer 可以被另一个 Reflection Probe Filtering 管线作为输入的一部分使用。可以认为他是 Cubemap 输入的其中一个面。

![move_subresource_example]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/move_subresource_example.png)

图：用反射举例资源Aliasing。

Move 操作就实现把 Lighting Buffer 资源赋值给 Cubemap 的其中一面。这会直接让 Deferred Shading 模块直接把输出写到 Cubemap 的面上，而不是创造一张单独的 Lightmap贴图。

而 Deferred Shading 模块在别处使用时，仍然允许分配独立的Render Target作为输出。

（我猜测，Setup阶段正常创建 virtual resource，在其后的compilation过程中将两个handle Aliasing。如果Aliasing失败，则正常创建不同的 concrete resources。）

### Frame Graph Compilation 阶段

编译阶段主要是针对 Frame Graph 进行优化和资源的分配工作。

- 裁剪掉所有的没有使用到的资源和 Passes：
  - 可以让声明节点（？Setup）的工作更随意一些。
  - 旨在减少配置的复杂度。
  - 简化条件 Passes，debug 渲染等。
- 计算资源的生命周期，根据使用情况创建GPU资源
  - 简单的贪心分配算法。
  - 在第一次使用前获取使用权，在最后一次使用后释放。
  - 为异步计算延长生命周期。
  - 根据资源使用情况推导 Bind Flags

可以把 Frame Graph 的数据结构抽象成一组数组。

```c++
struct resource_registery{
	resource[] resources;
};
struct render_pass {
	struct res_handle {
		int index;//in resource registery.
		int ref_count;	
	};
	res_handle[] resouce_handles;
    int write_resources_count;
};
struct frame_graph{
	render_pass[] render_passes;
};

//编译阶段会遍历所有的 pass 计算引用信息。
//所有的运行顺序都是在setup阶段确定的，编译阶段不会进行修改。
for(render_pass pass : frame_graph.render_passes) {
    pass.compute_res_ref_count();
    pass.compute_res_first_n_last_users();
    pass.compute_async_wait_point_n_res_barriers();
}
```

#### 裁剪算法说明

裁剪使用简单的 flood-fill 算法进行，从所有未引用的资源开始。

```c++
struct resource{
    int index;
    int ref_count;
    render_pass[] write_passes;
};
array resources = resource_registery.resources;
//首先先计算资源引用关系
//void compute_initial_res_n_pass_conuts(){
for(res : resources){
    res.ref_count++;
    for(pass : res.write_passes){
        pass.write_resources_count++;
    }
}
//然后根据引用剔除无用的pass
//void identify_unused_resources() {
stack resStack;    
for(res : resources){
    if(res.ref_count == 0)
        resStack.push(res);
}
while(res = resStack.pop()){
    for(pass : res.write_passes){
        pass.write_resources_count--;
        if(pass.write_resources_count == 0){
            //将所有引用了未使用资源的pass都剃掉
            //并且这些pass读取的资源也有可能需要去掉
            for(readed_res : pass.resources){
                readed_res.ref_count--;
                if(readed_res.ref_count==0){
                    //如果这些被读取的资源没有其他pass读取
                    //那么把写入资源的pass也去掉
                    resStack.push(readed_res);
                }
            }
        }
    }
}
```

用 debug 视图作为例子。为了方便，我们会直接在 Frame Graph 上增加线性化 Depth Buffer，用来检查深度渲染情况，但并不会一直开启。

如果Debug View 没有被用在最终的输出，那么编译阶段会把 Depth Buffer 和Debug View 都裁减掉，减少管线的复杂度。

![sub_graph_culling_example]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/sub_graph_culling_example.png)

图：compilation裁剪示意。

相反，当 Debug View 被开启，当做最终输出时。Lighting Buffer 是不需要被生成的，因此会用上述算法尽可能的裁减掉用不到的 Passes。

在没有Frame Graph之前，引擎需要确定哪些Pass需要被执行的时候是很麻烦的，也会在不同Pass之间引入耦合。现在 Lighting Pass 再也不需要知道 Debug 信息和相关Pass的存在了。

### Frame Graph Execution 阶段

执行阶段就是我们熟知的渲染流程了。遍历 Frame Graph 中没有被裁减掉的 Passes，并根据顺序调用他们的Callback函数。

在Callback函数里，就是直接调用 Render Context API了：设置渲染状态，资源，shaders，绘制命令。

需要注意的是渲染的时候，需要通过Setup阶段生成的虚拟资源句柄，取得实际GPU资源。

#### 异步计算

（没实操过这里看不太懂）

![execution_async_compute]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/execution_async_compute.png)

Kick Off：如果有 Render Passes 要求要异步执行，那么会有相应的同步点会一起加入到异步队列中。以保证写入完成之后输出资源才能被Consume。

虽然可以直接从 DAG 中推导出依赖关系，但是需要手调以获得更搞笑的执行效率。在GPU执行过程中，我们是不知道会有什么瓶颈。不太希望在使用大量带宽的绘制同时，执行那些带bandwidth-heavy的计算任务。

异步操作会让资源的生命周期变长，更多的资源会同时被激活。所以内存使用水平会提高，需要注意这一点。

![ssao_async_compute_example]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/ssao_async_compute_example.png)

图：SSAO 异步执行时，生命周期的变化情况。如果没有延长生命周期，资源可能会被Main Timeline 其他 Passes 重用。

启用异步计算的方法只需要在setup pass提供好信息即可。

```c++
AmbientOcclusionPass::AmbientOcclusionPass(FrameGraphBuilder& builder) {
	// The only change required to make this pass
	// and all its child passes run on async queue
	builder.asyncComputeEnable(true);

	// Rest of the setup code is unaffected
	// …		
}
```

现阶段 Frame Graph 是根据 Setup 的顺序执行的，但有可能未来会根据Profiling的结果，进行顺序的重排列。

### 用 C++ 代码来定义 Pass 的演示

有两种方式来定义 Pass：

**C++ Interface Class**

最初是用的是的这个办法，但是遇到一些缺点：

- 需要重复写很多相同的代码。
- 在 Setup-Compilation 之间，需要一点特殊的技巧来传递参数。
- class之间没有线性关系，看起来比较困惑。

**Lambda-Based**

所以后来采用了后者

- 保持线性的代码结构
- 移植老代码方便些。
  - 只需要把老代码包装在Lambda里，加上资源声明。
  - 一开始写一个超大的 lambda，再把Raw资源替换成Transients 资源，慢慢分解成小的（Pass）。
- 代价就是使用了大量的模板式API。

![pass_declaration_example]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/pass_declaration_example.png)

图：声明一个Pass 的示例代码。

由于Execute 会被延迟执行，所以Capture必须要用值引用。这么做需要注意两点：

- 当引用的数据是一个指针的时候，有可能会被提前释放。
- 有可能引用太大的结构数据，可以在编译的时候限制 lambda 的大小（到1kb）

### 渲染模块

![render_modules]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/render_modules.png)

stateless functions 就像上图演示的addPass一样。Persistent资源有会横跨多帧。

#### 模块间的通信方式

（音频没听明白）在老的架构里，模块间的信息传递主要是通过显式的输入输出来确定的。但是这会让引擎难以拓展新的数据结构，必须得修改函数签名。这不Scaled。

![render_module_communication]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/render_module_communication.png)

图：寒霜引擎采用了 Blackboard 的方式进行数据传递。

Blackboard 将数据通过组件的方式来进行存取，Pass可以通过在blackboard中加入局部数据，避免数据散播到全世界。

有时候有父子关系的模块之间也会在Setup时创建自己的blackboard进行数据传递，而不需要全局的的blackboard。

虽然 blackboard 很好的解耦了，

- 但是调用函数的时候，不知道需要传递哪些数据，需要看代码才能知道有哪些数据进行传递。
- 无法再编译阶段就验证无效的数据获取，运行期才会报告错误。

## Transient Resource System

**Transient** /ˈtranzɪənt/ *adjective*
 *只持续一小段时间；非永久的。Lasting only for a short time; impermanent.*

- 临时资源系统是 Frame Graph 的核心组件，支撑 Frame Graph 的运行。

- 临时的资源大部分使用在很少的Passes之间，生命周期短于一帧。
  - 这些资源主要是 Buffers，Depth/Color Targets，UAVs。
  - 尽量缩短资源的生命周期在一帧内。
- 只有在使用到资源的时候才分配。
  - 尽可能的快的释放释放。
  - Allocate Resources Directly in *leaf* rendering systems rather than somewhere Globally.
  - 让特性书写更加方便：可以写一些用到了很多资源的复杂任务，但只需要简单的在特性模块里声明资源的使用。

### 不同硬件平台的实现方式简述

![transient_resource_system_back_end]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/transient_resource_system_back_end.png)

图：对于不同的平台会使用不同的实现方式，对 texture 和 buffers也有区别。



![transient_textures_ps4]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/transient_texture_ps4.png)

图：PS4 的显存分配情况。x轴是运行的先后时间顺序，y轴是地址空间。

- PS4上来直接开一块很大的连续虚拟内存做池。
- 第一次使用贴图的时候分配虚拟内存块。
  - 用的通用 non-local 内存分配器
  - Patch or allocate GNM resource descriptors as needed
- 最后一次使用之后交换内存块。
- Commit physical memory to cover VA range used in current frame
  - 需要的时候增长物理内存池，并根据钱集镇的情况缩减到内存使用最高位。
- 资源在虚拟地址空间是会重叠的（overlap in virtual address space)，这个能被PS4 图形调试工具（Razor）识别。

![transient_textures_ps4]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/transient_texture_dx12.png)

图：dx12 的分配情况。

和ps4 不同的是，地址空间是不连续的，分为了好几段。所以没办法在不停止GPU工作的情况下任意增减内存池的大小。即使如此，重用内存仍然能让内存使用水平有显著的降低。

理论上，如果实现了全局分配的优化策略，可以预测到 Lighting Buffer 的情况，在AO 分配的内存的时候，直接重用Heap6 会节省更多。

> DX12 在资源堆上遇到的具体问题：
> Tier 1 的堆只允许有限类型（Buffer/Texture/RT/Depth Buffer）使用。需要创建更多独立的堆为不同的资源使用。
>
> 我们最常见的重用临时资源是RT或者DS，所以还不是太差。我们现在强制设置RT的Flag为临时的，不管用户有没有确认这点。
>
> Tier 2 堆更好一些，所有的资源都可以被重用，但仍然不理想。因为我们必须分配很多堆，然后在上面再细分（sub-allocate）出资源的空间。比起从一大块内存里分配，这样会产生更多的碎片化。
>
> 一旦资源被分配，就无法被移动，这意味着如果内存分配调度发生了变化，有些对象就必须重新创建以改变他们的位置。这可以通过caching d3d对象，通过重用来一定程度上绕开这个问题。
>
> 内存分配调度有可能会因为玩家的动作，分镜，UI发生变化。但也只是能用手数得过来的特殊调度，所以可以考虑用LRU cache。
>
> 这类问题在主机上不存在。Tiled 资源在未来会很方便 (XB1 memory aliasing类似)，however as of October 2016 there are significant CPU and GPU overheads to using them as RTVs / DSVs. 
>
> Additionally, resource heap tier restrictions prevent efficient tile mapping updates via CopyTileMappings. We sometimes want to use multiple heaps as page sources to back a single resource. Current UpdateTileMappings API can only take a single heap pointer, therefore multiple API calls are required.



![transient_textures_ps4]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/transient_texture_xbox1.png)



> Fragmentation-free dynamic memory allocation and aliasing
>
> Close to optimal ESRAM utilization automatically 
>
> - Don’t need contiguous memory blocks
> - Resources may be fully or partially in ESRAM
> - Overflow to DRAM when every ESRAM page is in use
>
> Hand-tune memory allocation based on profiling
>
> - Deny ESRAM for some resources
> - Allocate ESRAM top-down or bottom-up
> - Restrict ESRAM to % of the resource, place rest in DRAM
>
> Use a physical memory pool of ESRAM and DRAM pages
>
> Allocate all resources at unique virtual addresses
>
> - VirtualAlloc, CreatePlacedResourceX
>
> Allocate physical memory pages from pool on first use
>
> - ESRAM pages first, overflow to DRAM
> - Extend DRAM pool on demand
> - Shrink DRAM pool based on high water mark of last N frames
>
> Return physical pages to the pool after last use
>
> Update GPU page table before executing other commands
>
> - XB1-specific ID3D12CommandQueue API
> - Conceptually similar to CopyTileMappings 
> - Page table update happens on GPU timeline

由于个人经验不足，这一段没有翻译。接下来看明白再补充。

### 内存重用的考量

要非常小心：

- 保证资源的metadata state 的正确性。
- 要同时保证 Compute 和Graph 管线的正确性
- 要保证异步计算的正确性。
- 保证声明周期是正确的。这个做起来会比较麻烦，有时候需要保证cache里的数据被完全写入之后，才能被其他对象重用。

同时，为了保证正确性，也需要在分配的时候选用合适的操作。

- 针对新分配的资源使用 copy/clear。
- 需要保证resource在 Render Target or Depth write 状态。
- 要正确初始化metadata，和 fast-clear 差不多。但是内容保持未定义。
- 如果可能的话，优先使用discard 操作而非clear操作。

### Aliasing Barrier

由于不同的Render Passes 可能会用到同一块物理内存，所以需要在Passes之间加入同步点，防止pases之间互相覆盖内存数据。

- 加入必要的清空管线、缓冲的操作。
- 用更精确的barrier 来减少性能损耗。
- 在很困难的问题下，可以使用 **Wildcard** barrier 解决问题。
- 为了性能，Batch Aliasing Barrier and all other resoures barrier。

在GPU workload 之间加入同步点，aliasing barrier 和 resource condition barrier 很相似，只不过现在资源是会有复用的情况。当存在复用的资源的情况下，普通的resource barrier 就不够用了。从逻辑的角度而言，不同的workload在执行完全不同的内容，为了能为完全不同的逻辑资源建立连接，我们需要使用 aliasing barrier。

物理资源使用是有先后关系的，可能barrier之前的passes完成之后，需要暂停GPU的工作，显式Flush 缓存（make IHV cry）。

![aliasing_barriers]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/aliasing_barriers.png)

图：ps和 cs 并行运行，造成了大片的绿色messy。

这个例子中， 两种不同的shader在访问完全不同的逻辑资源，但实际上两个资源是做了aliasing。产生了竞争现象，变成这个鬼样子。

![aliasing_barriers]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/aliasing_barriers2.png)

图：Aliasing Barrier 和 Condition Barrier 类似，让不同的 work 顺序执行了。

有些时候这会影响性能。

### 内存 Aliasing 内存布局结果对比

#### 720p

![non_aliasing_memory_layout_720]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/non_aliasing_memory_layout_720.png)

图：也不知道是哪个平台，147MB。

![aliasing_memory_layout_720_dx12]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/aliasing_memory_layout_720_dx12.png)

![aliasing_memory_layout_720_ps4]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/aliasing_memory_layout_720_ps4.png)

![aliasing_memory_layout_720_xbox1]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/aliasing_memory_layout_720_xbox1.png)

图：三个平台差不多，都节约了47%左右的内存使用情况。

#### 4k

![non_aliasinyg_memory_layout_4k_dx12]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/non_aliasing_memory_layout_4k_dx12.png)

图：pc上没有 aliasing 的情况下，4k消耗1个G。

![aliasinyg_memory_layout_4k_dx12]({{ site.baseurl }}/images/2020-05-01-frame-graph-extensible-rendering-architecture-in-frostbite/aliasing_memory_layout_4k_dx12.png)

图：开启了 aliasing 的情况下，减少了1/2还多的内存使用。

## 总结

知道帧的全局的信息可以获得很多好处：

- 通过资源的 Aliasing 大量的节省内存：容易支持4k。
- 可以在有限条件下自定进行异步计算；
- 可以通过代码进行简单的来简化管线的配置，依赖裁剪和后续阶段进行优化；
- 还可以通过工具展现、诊断每一帧里潜在的优化点和错误。

图是一种非常有竞争力的描述管线的方式，从概念来说，和JobSystem一样，从直觉上对大家都比较友好。

另外Modern C++特性对保留模式的API设计很有帮助。

### 接下来的工作

- 通过全局信息，优化 Resource Barriers。
- 尝试用自动异步计算的能力，标记一些确定的 Render Passes。比如手动标记  Shadowmap Rendering Pass，然后可以自动的加入到异步计算。
- Profile 导向的优化工作：异步计算，优化内存分配。

希望有更多的引擎在未来可以采用类似的设计。因为这看起来像是最优的使用 DX12 或者 Vulkan 的方式。

### Q n A

- Q：如果标记了 Shadowmap Rendering Pass 在做异步Compute，和Graph任务混在一起怎么办？
  A：没办法，渲染错误。

- Q:多线程渲染怎么处理的？

  A：整个 Graph 的执行是线性的，单线程的。但是使用了 Shading System的 Depth/G-Buffer Pass是例外。Shading System 会产生很多Drawcall。在这些 Render Pass中，所有的Drawcall 都会产生到输出都是同一个 Render Target上，在 Pass 内是多线程的。

其他听起来很困难所以没有翻译，可以查查别人的（知乎上有）。



Frostbite / Electronic Arts, Yuriy O'Donnell, [FrameGraph: Extensible Rendering Architecture in Frostbite](https://www.gdcvault.com/play/1024045), GDC 2017, Video

Frostbite / Electronic Arts, Yuriy O'Donnell,[FrameGraph: Extensible Rendering Architecture in Frostbite](https://www.ea.com/frostbite/news/framegraph-extensible-rendering-architecture-in-frostbite), GDC 2017, Slider