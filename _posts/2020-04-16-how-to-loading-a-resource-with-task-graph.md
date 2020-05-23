---
layout: post
title: 使用 Task Graph 简化资源异步加载的复杂度
tags: task-graph multi-thread async-loading
---

# 使用 Task Graph 简化资源异步加载的复杂度

2020-04-16 由于任务全换人，现在无所事事。

2020-04-30 正在努力，争取在5.1 之前完成文档构建。



## Problem

大家好，今天主要来聊一聊多线程加载资源的相关思考和探索。

### 大量磁盘IO导致游戏假死

在网络游戏运行过程中，经常会遇到大量游戏资源同时加载的场景，典型的有：切换/进入新区域，需要加载装饰物资源；大量其他玩家突然出现在视野内，同时加载玩家缤纷各异的时装资源。此时资源加载过程需要进行大量的磁盘文件读取，也就是说我们常说的磁盘IO操作。

从经验上来说，耗时的 Disk I/O wait，会让我们的游戏出现“卡死”的现象。我们试着把场景描述的更精确一些：大量磁盘IO发生在主线程，导致了主线程的阻塞。由于早年的游戏大多数是单线程结构的，主线程阻塞后，会严重影响后续逻辑的运行。为了解决这个问题，我们需要缓解这类由于磁盘IO给主线程带来的阻塞的问题。

一般来说，单线程的循环执行伪代码就像下面这个样子：

```c++
while(GetMessage(&message)) {
    HandleMessage(message);
	Update();
	Render();
}
```

阻塞之后，无论是鼠标操作响应，还是渲染更新画面，都无法顺利执行，进入假死状态。

由于多人网络游戏的不可控因素（我们之前提到的典型场景是会在任意时间触发的），很容易导致卡顿频繁发生，影响游戏体验。现阶段业界的方法已经形成定式，比较明确的解决方案一般为以下几种：

1. 将阻塞的磁盘IO操作移出主线程，这样就保证了主线程不会阻塞了。
2. 在多线程加载的基础上，使用替代资源来避免资源“消费者”等待资源“生产者”的行为。
3. 预测使用的资源，并提前在空闲时加载缓存资源。这是一种空间换时间的策略。

在大部分情况下，以上三点是相辅相成的，需要互相配合来进行才能最大限度的避免游戏卡顿的现象。

我们今天主要专注在 *“如何使用合理的多线程模型来协助我们进行一部资源加载”* 这一主题上进行讨论。

### 多线程加载资源的常见方法

在深入讨论 Task Graph 之前，我想先和大家探讨一下潜在的解决方案。资源加载，我们可以认定为一种"生产-消费"行为。

#### 模型I：第一直觉

把*生产资源* 的加载逻辑完整移入到一个独立的 *加载线程* 里，是非常符合直觉的。我们将模型抽象成如下图所示：

![loading_basic_thread_model]({{site.baseurl}}/images/2020-04-16-how-to-loading-a-resource-with-task-graph/loading_basic_thread_model.png)

图：直觉驱动的加载结构。左侧为单线程加载的逻辑，右侧为改造后的逻辑。

在这个模型中，加载逻辑都同一个线程执行，通过主线程的入口、出口进行数据交换，避免了大量的同步交互。从代码层面的考量，在已有系统上做改动的前提下，并行化修改的范围也比较小。但是简单的模型在适应性上是存在缺点的：

1. 并行阶段只存在于*“加载-->完成”*这个阶段。主线程更新先于加载完成的情况下，从宏观角度而言，会退化成单线程一样的模型。
2. 在 Loading Thread 里，仍然会发生磁盘IO阻塞线程的情况，浪费了线程的计算资源。

当然，`问题1`可以采用默认资源（*“简化的替代资源”* ）进行回避。`问题2`是存在改进空间的，我们设法让磁盘IO和计算资源同时被利用上，构建了流水线模型。

#### 模型II：流水线（Pipeline）

主要的改进思路就是让磁盘IO的阻塞控制在最小的范围内，不要影响其他逻辑的运行。

以此目标我们提出了流水线结构模式，将一个加载任务划分成了多个阶段（Loading Stage/Parsing Stage/etc），并针对每个阶段分配独立的线程。我们将线程模型改造成如下形式：

![loading_pipeline_thread_model]({{site.baseurl}}/images/2020-04-16-how-to-loading-a-resource-with-task-graph/loading_pipeline_thread_model.png)

图：根据Stage分配线程来完成加载任务。

和之前相比，我们将 `Loading Thread` 拆分成了两个线程： 读取文件的`DiskIO Thread` 和 解析资源的`Parsing Thread`。让Disk IO wait只影响读取阶段，不影响解析阶段。

我们可以设想一下这个模型运行时的数据流向：当前我们有2个资源在队列里等待加载，当第一个*资源A*的磁盘阻塞完成后，流转进入 Parsing Stage，此时 Parsing Thread 处理*资源A*，而DiskIO正好被空出来给*资源B*进行加载。正如下图展示：

![/pipeline_runtime_demo]({{site.baseurl}}/images/2020-04-16-how-to-loading-a-resource-with-task-graph/pipeline_runtime_demo.png)

图：流水线模型加载4个资源时，每帧数据流向展示。

到这一步的位置，我们认为已经能满足大部分的加载需求了。在这个模型里，Stage是可根据加载的逻辑进行伸缩的，线程同理，据此我们可以针对不同引擎的加载的步骤进行快速调整。

#### 流水线加载的缺点

然而，作为一个通用的解决方案，流水线的缺点也非常明显。在实际应用中会遇到如下问题：

##### 不同的加载逻辑，划分的Stage差异过大

资源种类太多，加载逻辑差异过大的时候，流水线无法合理特化成不同的模型。

我们需要根据所有资源加载逻辑进行精心设计 Stage。必要时为极少数资源种类增设额外的Stage，而大部分资源需要跳过。这会给加载逻辑的构建带来很大的麻烦。

1. 大部分资源会出现某个Stage 是空操作的现象，让程序员对业务理解带来一定的障碍。
2. 新的资源类型需要改造管线时，加载逻辑差异过大时，会面临 *改管线 or 改逻辑*  的挑战。

##### 线性加载过程导致顺序固定

流水线的天然特征，就是顺序规则是明确的，不太适合在调整管线的内请求的优先级。这对于引擎内的希望的优先级功能，在实现层面上要面对数据同步的挑战。

而另一个方面，流水线表达的一个是线性的加载模型。现代引擎的资源设计里，资源往往会有许多依赖项，这是难以避免的现象，我们不太可能通过设计规则来避免这个现象。

1. 大部分情况下加载的热点出现在 Disk IO 线程的阻塞上，当IO耗时不均匀时，一个巨大的文件就会堵住所有队列资源的加载过程。而IO后续的其他 Stage Thread会被闲置。
2. 实际开发经验中，我们无法假设每一个资源都是正确加载的。为了支持失败状态之类的分支情况，我们不得不在管线的每个Stage里增设特殊的开口处理规则。

##### 对加载逻辑之间的依赖不友好

当一个资源的当依赖项过多的情况下，后续步骤需要等待前序步骤完成。有依赖关系的任务，不太容易进行线性的拆分。

让我们用材质资源的例子，来说明解决资源依赖的问题，是如何影响并发加载问题的。设想一下在c++编码过程中，我们书写材质加载的逻辑，应该如下图左侧所示，是一个线性向下的结构。但实际上，在我们将图中的依赖关系用线连起来之后，可以展开成右侧的一个依赖图。

![material_loading_routine]({{site.baseurl}}/images/2020-04-16-how-to-loading-a-resource-with-task-graph/material_loading_routine.png)

图：将线性的步骤展开后可以得到一个DAG，着色指的是流水线的线程。

用下面这个数据流向表格想象一下：A材质加载完成的标志：依赖于BCD三个贴图的加载完成。

> 流水线的加载实例：
>
> 资源A：材质      <-----------------------------+
>
> +----＞资源B：贴图0                               |
>
> ​             +---->资源C：贴图1                    |
>
> ​                         +---->资源D：贴图2 -----+ （完成这项子任务后，资源A的依赖完成，进行回溯）

流水线资源A是无法从流水线中移除的。这种情况下，流水线模型中就死锁了，需要做特殊的退出处理。如果按照流水线的方式，是无法完成这个图的任务的。

我们解决这种情况的思路是：假设A材质加载完成移出流水线，再等待BCD完成之后，将 Assemble Stage，额外的抽离出来在流水线之外（主线程）完成。在实际操作中，每一个资源在加载完成后仍需要判断是否“真正的完成”，等待依赖资源的完成都是非常麻烦的，需要很巧妙的技巧去包装。

除了资源加载，我们在其他领域亦会面临着大量的“半完成”状态，此处不在赘述。更多的关于依赖的示例说明，可以参考这篇GDC视频：[Job Graph: Task Graphing in Mortal Kombat][job-graph-first-glance-n-explained] ，视频里将依赖关系解说的非常清楚。

#### 多线程加载关键需求总结

至此，我们可以说是接下里的思路都可以整理清晰了：

- DAG图，是能很好表达依赖关系的。
- 按照这个依赖关系先后顺序执行，可以避免在运行过程中，需要等待依赖完成的情况。（每一个步骤开始时，前序步骤已经完成。）
- 我们要构建一个多线程模型，来有序执行这个依赖关系的图。
- 这个多线程模型必须是能将数据同步变得尽可能的简单。
- 这个多线程模型能有效的使用硬件资源，最好能在阻塞的条件下，填满其他可用的资源，避免资源浪费。
- 我们需要这个组件能支持动态分支的能力。



## Context

到上一步为止，我们使用DAG描述了每个任务之间的依赖关系。这和我们今天讨论的目标很接近了，Task Graph 是什么？我觉得伯克利学院的这个课件会给你一些最直接的认识：[ Concurrent Execution Patterns : Task Graph][Concurrent-Execution-Pattern-Task-Graph]。

### 什么是 Task Graph

在本文中，Task Graph 指代的是一整个并行计算的解决方案，包含了：

- Tasks and Graph：描述依赖关系的DAG，正如上文图里展示。我们认为每一个节点，都是一个任务，并把节点之间的有向连线，称为节点之间的依赖。
- 任务调度：运行时模块，负责将任务根据顺序映射到具体平台的执行单元上。在本文中，执行单元指的就是线程模型。
- 线程模型：用来并行执行任务的模块。

三者构成了 Task Graph 的解决方案，主要为了解决有严重输入、输出依赖的任务运行，和尽量提高运行效率而形成的一种执行模式。

> 在计算问题领域，有很多问题可以分解成一组原子任务（Atomic Tasks），这些 Task 之间，会将计算结果作为输入输出进行传递，而形成了依赖关系。
>
> 由这一组任务和基于参数的依赖组成的DAG，就是Task Graph。当每一个任务完成之后，任务停止的同时将计算结果视为输出，作为后续任务的输入参数。
>
> Task Graph 有可能会在编译器就能决定顺序，也有可能要到Task Graph执行的前一刻才能完成有执行度的先后顺序排列。当然，大部分时候，Tasks的数量是需要在执行期间，根据参数发生变化的。

##### 清晰的数据访问

通过任务来区分对某一个数据的读、写逻辑。前置任务写入之后，数据是不会被再被修改的，我们认为下一个任务执行时可以安全的访问数据。

##### 判断任务开始的条件

任意 Task，必须在它的前置任务全都完成之后，才会认为可以安全执行。当一个任务完成时，会主动通知后续任务它的其中一个前置任务完成了。如下图右侧所示。

![task_graph_pattern]({{site.baseurl}}/images/2020-04-16-how-to-loading-a-resource-with-task-graph/task_graph_pattern.png)

图：task graph，引用自 [ Concurrent Execution Patterns : Task Graph][Concurrent-Execution-Pattern-Task-Graph]

##### 如何判断结束状态

当图中所有的任务完成时，认为整个计算过程结束。实际情况下，有可能会有多个完成节点（没有后置任务的任务）的存在。我们可以根据依赖关系，认为那些没有后置任务的任务完成时，计算过程结束。如上图所示， C、E、F三个任务完成时，图完成。

##### 条件依赖

注意到，在图中的C节点，是一个条件执行的状态。条件主要是用来支持执行错误，任务取消，任务未完成等特殊状态变化。

#### 线程在实现中扮演的角色

Intel 在 GDC 2010 Talk [Task Based Multithreading - How to Program for 100 cores][intel-task-graph]，阐述了线程模型如何与 Task Graph 协作完成计算工作。我将视频中最重要的一张图重新整理如下：

![task_graph_thread_model]({{site.baseurl}}/images/2020-04-16-how-to-loading-a-resource-with-task-graph/task_graph_thread_model.png)

图：Task Graph 使用的线程模型。分为 Task Graph 和 线程模型两个部分。

##### 线程模型与 Task Graph 调度逻辑

DAG 构造好之后，可以遍历出没有依赖的 Task，送入到线程模型的 Work Queue中。而其他有依赖的任务会被暂时不做处理。

Thread Pool 会从Work Queue中取得任务，并为每一个空闲的线程分配。当队列中的任务执行完的时候，会通知它的后置任务。

这些后置任务在每次收到通知的时候，检查前置依赖情况。当所有前置任务完成（引用计数清空），则会加入Work Queue等待被执行。

##### 任务间数据共享

从语言层面考虑 Task Graph 有两种数据传递方式：

1. 为每个Task 定义返回值，输入参数，将上一个任务的返回值作为下一个任务的参数进行交互。
2. 将数据放在同一个共享数据里，并将Read/Write操作区分在不同的Task里。并设置依赖。

作为依赖基础，Task Graph 要求同一个数据不能同时被两个 Task 访问。换句话说无论通过哪种数据共享方式，在执行过程中都可以安全的访问数据。

#### Task Graph 的特点

**业务和程序结构解耦**：Thread Model 不再关注业务的结构，每个Thread只需要关心执行Task即可。这让业务流程和程序结构完全解耦了。个人觉得这就是 Task Graph 最关键的好处。线程模型是统一的通用模型，在不关心业务结构的情况下能最大程度的支持不同变化的业务。

##### 业务易于理解

而程序员在构造加载流程的时候，可以聚焦在数据的流转上。图自身的表达能力是足够的。不再被实现上的线程同步的噪音干扰。

##### 有效利用硬件资源

任务在加入到Work Queue 的时候，已经明确了不在需要等待其他的任务，避免了等待行为。可以认为没有等待的情况下，任务很方便的能填满线程。

##### 简单的同步操作

由于数据的读写，被拆分在不同的任务上。同步行为就变成了任务的依赖。在实现层面上，会使用一个 `atomic<int> ` 来记录前置任务的数量，当数值变为 0 的时候就任务前置任务被清空。这是几乎所有的 Task Graph 惯用的手法，原子级别的信号通知让效率提高了不少。

#### 各种 Task Graph 的接口使用简洁

市面上成熟的库很多，我自己有认知的有两个比较著名的：

1. 一个是大名鼎鼎的 Intel Task Thread Blocks(TBB)，如果需要在c++层面上做集成，这应该会是一个好选择。
2. 另一个是.Net framework 4 Task Parallel Library (TPL)，如果有幸在C#语言上工作，TPL也不错。我的同事向我推荐这个framework。
3. 除此之外，Unreal4 也有一个正在使用的 Task Graph。

实在不知道怎么介绍接口，我就用三个 TPL example 来说明吧。

##### 如何构建基本的任务和依赖

**New**：正常情况下，用户可以通过`static Task Task.StartNew(callback)` 的方式来创建任务。在任务创建完成后，TPL会立即将Task加入执行队列。

```c#
var t = Task.StartNew( () => {
    DateTime dat = DateTime.Now;
    if (dat == DateTime.MinValue)
        throw new ArgumentException("The clock is not working.");
    if (dat.Hour > 17)
        return "evening";
    else if (dat.Hour > 12)
        return "afternoon";
    else
        return "morning";
});
```

**Fork**：同时，为了创建以来，我们可以直接在Task对象上调用成员函数`Task Task.ContinueWith(callback)`来创建依赖任务。由于前一个任务有可能已经执行完成，所以创建好的依赖任务也有可能直接运行。

值得指出的是，TPL提供了模板接口定义返回值，Task之间可以通过返回值传递参数。如下代码所示：

```c#
var c = t.ContinueWith( (antecedent) => {
    if (t.Status == TaskStatus.RanToCompletion) {
        Console.WriteLine("Good {0}!", antecedent.Result);
        Console.WriteLine("And how are you this fine {0}?", antecedent.Result);
    }
    else if (t.Status == TaskStatus.Faulted) {
        Console.WriteLine(t.Exception.GetBaseException().Message);
    }
});
```

运行状态和依赖的关系，主要有库来维护。我们无需担心状态问题。如果有需要，可以针对同一个Task对象调用多次`ContinueWith()`函数来构造 fork 形态的图。

**Join**：为了能支持“创建即启动”这个目标，需要在创建过程中传入依赖Task。TPL提供了`Task Task.ContinueWhenAll(task_list, callback)` 来完成这项工作。如果用材质的接口来表示的话，应该写成如下形式：

```c#
Task readMaterialFileDataTask = Task.Factory.StartNew(arg =>
{
    MaterialTaskGroup argTaskGroup = arg as MaterialTaskGroup;
    argTaskGroup.ReadMaterialData();
}, taskGroup);

Task parseMaterialFileDataTask = readMaterialFileDataTask.ContinueWith(
    (antecedent, arg) =>
{
    MaterialTaskGroup argTaskGroup = arg as MaterialTaskGroup;
    argTaskGroup.ParseMaterialData();
}, taskGroup);

Task parseUsedTextureTask = readMaterialFileDataTask.ContinueWith(
    (antecedent, arg) =>
{
    MaterialTaskGroup argTaskGroup = arg as MaterialTaskGroup;
    argTaskGroup.ParseUsedTextureDatas();
}, taskGroup);

taskGroup.finalAssmebleTask = Task.ContinueWhenAll(
    new List<Task> { parseMaterialFileDataTask, parseUsedTextureTask }, 
    (antecedent, arg) =>
{
    MaterialTaskGroup argTaskGroup = arg as MaterialTaskGroup;
    argTaskGroup.AssembleTexutureIntoMaterial();
}, taskGroup);

Task.WaitAll(taskGroup.finalAssmebleTask);
```

**Wait**：在上边的例子中已经提供了等待的接口。拥有了 create/fork/join/wait 的接口之后，我们就拥有了创建基本的 task graph能力。

p.s. 一般来说，创建立刻执行的任务，比较适合计算性的任务。在某些创建和执行得分开的特殊场合， 也可以使用`staitc Task.Start(callback)` 的方式，来进行Task 的构建。

##### 如何动态构建子任务和依赖：Master & Worker

**Compile Static, Runtime Static, Runtime Dynamic**

在构建 task graph 的过程中，我们有可能需要区分静态 graph 和动态 graph。

- 编译期静态：指的是我们在代码里明确了图的结构，在代码编译之后，图就已经被构造成功。我们在上文中举例的流水线模型，就是一种典型的编译器静态。这种方式是没有运行时消耗的，难点在于如何设计出静态构建图的接口。
- 运行期静态：指我们通过编码的方式，在运行时创建 graph，这种方式是需要承担构建图的消耗代价的。但是图本身的结构，已经被代码的表达所固定。
- 运行期动态：在某些条件下我们没办法预先知道图的结构，需要根据环境一边运行一边决定。在这个条件下我们需要动态构建图的节点。

为什么我们需要运行期动态构建图呢？我仍可以使用材质加载的例子进行说明。

![dynmaic_create_child_task]({{site.baseurl}}/images/2020-04-16-how-to-loading-a-resource-with-task-graph/dynmaic_create_child_task.png)

图：材质依赖于贴图加载。

如图所示，在左图红框里，表示的是材质需要依赖贴图的加载完成。但是我们需要几张贴图，是依赖于上一个`Parse Used Textures` 任务完成的，我们必须完成Parse，才能确定需要构建的加载贴图的Task数目。

为了支持这种情况，TPL允许特殊的构建选项，通过 `CreationOptions.AttachToParent`，我们可以为某一个父任务创建很多个子任务。这些子任务完成之后，父任务才被通知为完成状态，并向后传播。

从实现层面来说，父任务实际上已经完成了执行过程。所以更类似是等待完成的功能。UE4 提供了  `Task.DontCompleteUntil(sub_tasks)` 接口，完成Task后不直接通知后续任务，而是等待子任务完成。从结构上，我们可以想象成上图右侧的结构。

##### 如何在递归里使用 Task Graph ：Divide and Conquer

在某个任务中，我们需要构建子任务，并等待运行结果返回后继续执行。一般这类型的构建，会出现在递归执行的算法中，比如斐波那契数列、Merge Sort等。

由于等待子任务完成，会打破每一个任务不需要等待的约束。这会使得执行Task的硬件资源进行等待。当类似情况占满之后，会出现死锁现象。为此，库的设计需要能够主动 pending Task的功能。由于资源加载不需要此类方法，所以没做深入研究。在这里提一句有这么个东西，大家如果感兴趣自己去看吧。

### 如何构造并使用 Task Graph 加载资源

so，了解到 task graph 的基本状况之后，我们来探究一下，如何应用在资源加载上吧。

#### 针对游戏特化线程模型

从文章最初开始，我们要解决 Disk I/O wait 导致等待的问题。在并发条件下进行磁盘读取的情况下，并发数量的增加会导致IO效率大大降低，所以我们有意识的针对磁盘IO的构建一个独立运行线程。除了独立运行的IO Thread之外，可以活用剩下的硬件资源作为通用的 Work Thread。

![destiny_thread_model_ps3]({{site.baseurl}}/images/2020-04-16-how-to-loading-a-resource-with-task-graph/destiny_thread_model.png)

图：Destiny Engine 使用的线程模型。还不知道为什么没有单独设立IO Thread。

#### 特化 Scheduler 用以支持加载特性

在 TPL 中，我没找到可以定制 Thread Pool 的位置。根据MSDN的描述（https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskscheduler?view=netcore-3.1），我构建了一个自定义的Scheduler，这个Scheduler每次只会执行一个任务。并把这个Scheduler应用在所有IO任务上，用来保证 IO 任务的互斥。

我们为 Scheduler 配好三个不同的队列，用来表示高中低任务的要求。当任务加入Scheduler的时候，会根据任务的属性分别加入对应的队列里。

#### 如何分解加载任务，构造Task Graph

本质上来说，task能解决的问题只是尽量吧阻塞控制在小范围之内。task拆分实际上是一种空间换时间的方法，当有多个任务在执行过程中，会消耗更多的内存。我们需要根据资源的流程，把 I/O，解析步骤分开，并尽可能的在不互斥的情况下进行同步。

当然 Task Graph 也有应用上的局限性，对于边读边解析的加载任务，并不能很好的拆分 Task，即使强行拆开，也因为依赖关系，退化成和线性执行的流水线类似。如果引擎中的任务都类似此种情况，不太建议使用 Task Graph 进行改造处理。

##### 原则第一点：不要有循环重入的 Task。

Task Graph 是一个 有向无环图（DAG），DAG是天然就能避免死锁的。因此我们在构造 Task Graph 的时候，需要注意的避免的是循环依赖。

##### 原则第二点：从数据的访问来分析。

实际上无论使用哪种方式，我们在拆分任务的时候，都需要对数据进行考量，将读写区分开来，特别是针对硬件资源的RW操作。避免并行的任务里因为出现数据竞争而造成错误。

##### 原则第三点：考虑任务的粒度

一个复杂的任务，可以在不同的 Level上区分成不同数量的小任务。如果任务的数量过多，会导致线程长期处于硬件资源抢占的状态导致性能下降；而任务数量过少，则无法充分利用硬件资源。

一般情况下，我们需要考虑硬件水平和线程模型。大部分的游戏引擎，线程数量是和硬件核心数目挂钩的。在Bungie的视频里，不同平台（xbox1/ps4 ）随着时代的发展，线程数量是一致的。

合适的任务数量是速度的保证。

##### Example

用C# 写了一个模拟加载材质的示例代码，已经在提交到github上了。

#### Graph Profile 工具

从实际的经验上来说，摸黑调试 Task Graph 是非常不明智的。在很多技术分享中，都提到了 Profiler 工具的重要性，鉴于篇幅有限我就不在此赘述了。我在这里给一个我用 Javascript 写的数据展示器的截图。

![task_scheduling_profiler]({{site.baseurl}}/images/2020-04-16-how-to-loading-a-resource-with-task-graph/task_scheduling_profiler.png)

图：每个线程的调度信息展示。

#### 性能测试与调试

##### Mock数据

|    测试内容：    | 线性加载 | taks graph |
| :--------------: | :------: | :--------: |
|   **理论数据**   | **9100** |  **6800**  |
|     Mock 1st     |   9122   |    6620    |
|     Mock 2nd     |   9124   |    6615    |
|     Mock 3rd     |   9121   |    6615    |
|     Mock 4th     |   9124   |    6616    |
|     Mock 5th     |   9129   |    6612    |
| **Mock平均耗时** | **9124** | **6615.5** |

##### 实际的图片测试

|    测试内容：     |   线性加载   | taks graph  |
| :---------------: | :----------: | :---------: |
| Image Loading 1st |   17.91 ms   |  10.62 ms   |
| Image Loading 2nd |   17.64 ms   |  11.28 ms   |
| Image Loading 3rd |   16.62 ms   |  10.57 ms   |
| Image Loading 4th |   17.91 ms   |  11.33 ms   |
| Image Loading 5th |   17.09 ms   |  12.08 ms   |
|   **平均耗时**    | **17.43 ms** | **11.18ms** |



## Conclusion

在本文中，主要是介绍了 Task Graph 的优势，和如何用来加载资源的思路。在后续过程中，还会探究分享如何支持任务取消、调整优先级的功能。

当然除了用于资源加载， Task Graph 还有很多可能性。现阶段，已知很多游戏引擎，都开始了将游戏逻辑拆解成  Jobs 式的编程方式。UE4 更是直接用 Task Graph 构建了多线程渲染的基础。

如果有机会我们下次再深入探究其他的问题吧。



## References & Further Reading

[Concurrent-Execution-Pattern-Task-Graph]: https://patterns.eecs.berkeley.edu/?page_id=609	"Concurrent Execution Pattern: Task Graph"
[intel-task-graph]: https://www.gdcvault.com/play/1012321/Task-based-Multithreading-How-to "task-based-multithreading-how-to-program-for-100-cores"
[msdn-csharp-task-class]: https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task?view=netframework-4.8	"MSDN上对C#语言的Task的描述"
[msdn-csharp-task-programming]: https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/task-based-asynchronous-programming?view=netframework-4.8
[job-graph-first-glance-n-explained]: https://www.gdcvault.com/play/1017770/Job-Graph-Task-Graphing-in "Job Graph: Task Graphing in Mortal Kombat"
[task-graph-using-in-game-bungie]: https://www.gdcvault.com/play/1022164/Multithreading-the-Entire-Destiny "Multithreading the Entire Destiny Engine"
[dataflow-tpl-library]: https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/dataflow-task-parallel-library
[granularity-and-parallel-performance]: https://software.intel.com/zh-cn/articles/granularity-and-parallel-performance


1. [Concurrent Execution Pattern: Task Graph，https://patterns.eecs.berkeley.edu/?page_id=609 ][Concurrent-Execution-Pattern-Task-Graph] ，这是伯克利大学eecs专业的课件，文章是认真构建的，描述的非常清晰。最近被墙了，需要梯子。
2. Microsoft，MSDN，[Task Class，https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task?view=netframework-4.8][msdn-csharp-task-class] ， 白暖暖推荐的：由于MSDN除了描述了语言之外，还附带了很多例子关于 Task Graph 使用的用例，所以对于构建一个基础组件的API 设计、能力呈现是一个很好的解说。
3. Microsoft，MSDN，[https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/task-based-asynchronous-programming?view=netframework-4.8][msdn-csharp-task-programming]，同上
4. NetherRealm Studios， Gavin Freyberg，[Job Graph: Task Graphing in Mortal Kombat，https://www.gdcvault.com/play/1017770/Job-Graph-Task-Graphing-in ][job-graph-first-glance-n-explained] ，GDC 2013，这是游戏内使用 Task Graph 很好的介绍，用来展现 Task Graph 能够解决的问题。
5. Intel，Ron Fosner，[Task Based Multithreading - How to Program for 100 cores，https://www.gdcvault.com/play/1012321/Task-based-Multithreading-How-to][intel-task-graph] ，GDC 2010，是 Intel 的广告，前半部分把 task graph 和线程模型做了详细说明。
6. Bungie，Barry Genova，[Multithreading the Entire Destiny Engine，https://www.gdcvault.com/play/1022164/Multithreading-the-Entire-Destiny][task-graph-using-in-game-bungie]，GDC 2015，这是 Bungie 把 Task Graph 应用在游戏上的例子，同年还有娜姐的多线程渲染，亦推荐阅读。
7. Microsoft， MSDN，[https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/dataflow-task-parallel-library][dataflow-tpl-library]，在学习 TPL 的过程中发现的一个特殊的组件。感觉更适合用在资源加载上。但是没有进行深入的探索。
8. Intel， [https://software.intel.com/zh-cn/articles/granularity-and-parallel-performance][granularity-and-parallel-performance]， 2016.10.11，粒度与并行性能。
