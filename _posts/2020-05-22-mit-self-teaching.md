---
layout: post
title: 借助 MIT 网站进行自我提高
mathjax: true
---

# 借助 MIT 网站进行自我提高

2020.02.05 中国正处于冠状病毒爆发期

2020.05.22 闲聊之余意识到，完成本文可能会对别人有一些帮助。

这是一篇重新整理的旧笔记，希望能把我查询过程和线索分享给朋友，避免重复介绍。同时我也需要梳理过去的思路，帮助我补齐现阶段“自救”的前期工作。

全文是按照信息收集的顺序进行的，所以结论都在文章末尾。

## 为什么要通过 MIT 网站进行自学？

我想，在回答这个问题之前，应该先回答：*”为什么要自学？“* 

出发点很简单，我写两个公式，初学渲染都会涉及渲染方程：
$$
L_o(\mathbf{x}, \omega_o,\mathbf{\lambda},t) = L_e(\mathbf{x}, \omega_o,\mathbf{\lambda},t) \\
    + \int_\Omega
	{
		f_r(\mathbf{x}, \omega_i, \omega_o,\mathbf{\lambda},t)
		L_i(\mathbf{x}, \omega_i, \mathbf{\lambda},t)
		(\omega_i \cdot \mathbf{n})
	}
	d{\omega_i}
$$

其中的半球积分  $$ \int_\Omega $$ 应该是拦住大部分人的第一个符号。所以我不得不捡起微积分的内容进行学习。随着学习的深入，我们发现无法通过计算对积分进行求解，于是借助蒙特-卡洛方法求取数值近似解。此时我们就接触到第二个公式：



$$
L_{pixel} = \int_{pixel area} L(x_p) dA \\
\approx {1 \over N } \sum_{n=1}^N L(x_n),
$$

我想写到这里，已经能够回答*“为什么要自学”*这个问题了，而且自学方向亦已明确。那么我该如何安排我的自学呢？这就是另一个问题了。

### 自学的基础和准备

从长远来开，只需要做到精力管理、安排计划、长期坚持即可。引用万维刚的“自学”的学问，我们自学的基础来自于如下两个基石：

* **自教**：自教是自学的前提，得有点自主安排的能力才行。
  * 有能力为自己选择合适自教材。
  * 能够制定出合适自己的有效目标。
* **自信**：要学会清理自己的内心，做好自我管理。
  * 面对威胁时，心理上会进入封闭状态，无法很好吸收知识。

从执行层面来说是可以遵循明确执行路径的：

1. 收集质量优秀的教学材料。
2. 整理、组织正确的学习内容。
3. 安排、制定学习计划。
4. 养成良好的生活和学习习惯，避免拖延。
5. 整理学习成果，并定期输出个人思考总结。
6. 要完成自我考核、监督。同时评估自己的身体，精神状态。

从路径中不难发现，我们几乎就是模拟在校学生的学习生活。从最后的考核来看，反馈形成了完整闭环。我想应该要相信许多学校都采用的教学方式。（由于之前缺乏了反馈闭环，所以学习成果不显著，在此特别指出。）

### 所以，为什么是 MIT 网站？

#### Open Courseware

公开课网站 [MIT OpenCourseWare](http://ocw.mit.edu/index.htm)，是我见过的**完整度最高**的网站。课程丰富全面，在国内也有很广流传度，可以轻松找到汉化资源。

网站将课程按照学院进行了分类，也可以通过课程号快速查询。甚至同一门课程会有多个年份的视频提供参考。（Prof. Gilbert Staring 老爷爷出新视频啦！）经小伙伴提醒，课程号查找框是 google 的，**不翻墙看不见查找框**。

每一门课程的主页，提供了质量极高的教学材料。以[微积分](https://ocw.mit.edu/courses/mathematics/18-01-single-variable-calculus-fall-2006)课程为例，你能在课程主页中找到：

* 教学大纲：教学时间计划、前导课程、学习目标、教科书阅读指南。
* 教学内容：视频材料，课程关键问题和作业的详细解答步骤。
* 考核内容：课后作业习题集，考试复习大纲，和考试试卷。

基本上，只需要通过网站，就可以完成收集材料、制定计划的工作。换句话说，直接了满足**“自教”**的前个三步骤，省去了大量的时间。

#### OCW-Scholar Version

需要特别指出，OCW 有部分课程会被标记为 [OCW-Scholar Version](https://ocw.mit.edu/courses/ocw-scholar/)，这些课程为了能够让读者进行跨专业的学习，进行了非常细致的教材设计工作，视频解说比其他版本更为详细。

这类课程是针对终身学习的读者而制定，提供了不需要 Textbook 也能学会的教学材料。这些课程 ，涵盖了大多数重点基础学科，诸如微积分。因此在不同版本的公开课前，我会优先选择 OCW Scholar Version。

作为一个受益者，在此衷心的感谢 Stanton Foundation 的赞助。

## MIT 网站信息汇总

前文提到，我当下需要学习微积分和统计学。但是数学作为一个工具，永远需要应用在具体的领域上。那么完成学习之后，后续有哪些可选的课程呢？我该如何安排我整体学习计划呢？

### MIT 网站漫游

我把自己假设成一名刚入校的大学生，此时我最关心的应该是毕业前，学习的课程列表。

#### EECS School

MIT OCW 的网站没提供具体信息，所以我的思路是去计算机学院看看培养计划。网站以学院分类的主页，提供了不同院系的链接 [OCW Department](https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/)。

虽然经过几年的改版，[电气工程和计算机科学学院(EE&CS)](http://www.eecs.mit.edu/) 网站仍然只能提供非常少量的信息。我只能在最显眼的位置找到关于课程的[Road Map 培养计划](https://www.eecs.mit.edu/docs/ug/freshman_roadmaps.pdf)和毕业学分Checklist（不是很重要）。

虽然Road Map涉及了课程和课号，但并不是很详细，无法帮助我完成 Checklist 的填写。我觉得需要的是一份更清晰的必修课清单。

#### MIT Course Catalog

得益于2018年的网站更新，现在我们可以在学院主页上找到所有学校的课程分类了。[CourseCatalog Degree Chart](http://catalog.mit.edu/degree-charts) 完整列出了每个专业毕业 Requirement。

基于要求，学习计划主要分为了两部分：

* **GIR** - General Institute Requirement。主要包括数学、物理、化学、生物、体育。当然也包括人文艺术社科和沟通要求。[http://catalog.mit.edu/mit/undergraduate-education/general-institute-requirements/](http://catalog.mit.edu/mit/undergraduate-education/general-institute-requirements/)
* **Departmental Program**：学院的课程计划，根据递进关系包括学院的公共必修课、CS专业课程、高级本科科目，和自主探索科目。[http://catalog.mit.edu/degree-charts/computer-science-engineering-course-6-3/](http://catalog.mit.edu/degree-charts/computer-science-engineering-course-6-3/)

**非常有帮助的是**，网站上的每一个课程号点开后，会弹出一个气泡窗口，提供了每一门课程的简介、开学时间，前置课程、和教学时长，让我们更容易勾勒出课程安排的全貌。

> 早些年浏览学院网站的时候，一直不明白*电气工程和计算机科学（6-2）*，和 *计算机工程和科学（6-3）* 两个专业的区别在哪里。找了好久毫无头绪，顺便这次也在新版网站得到了答案。我们可以通过*辅修学位（Minor in Computer Science）* 的课程列表里看到明显的差异。
>

### 整理网站信息

我的目标是专注在图形学的学习上，所以我在这里选择的是 6-3 的专业进行资料的索引。**值得一提的是**，阅读一下数学系（18-C）的课程设置会获得很多帮助。

#### 术语对照表

由于资料里会有很多缩写，每次都会忘记。我在此帮助大家整理一下术语表：

* SB Degree - Bachelor of Science (in Mathematic) 理学学士学位

  * 18=Mathematics，参见 [Department of Mathematics](http://catalog.mit.edu/schools/science/mathematics/#undergraduatetext)
  * 18-C 是 Bachelor of Science in Mathematics with Computer Science

* B.E, B.Eng,  B.A.I - Bachelor of Engineering 工学学士学位

  * 6=EECS，6-3 是计算机工程和科学专业，参见 [Department of EECS](http://catalog.mit.edu/schools/engineering/electrical-engineering-computer-science/)
  *  B.A.I 是拉丁名字：*Bachelor in the Art of Engineering*

* 课程编号说明：

  * 课程编号 ：`DepartmenID.CourseID`。`6.xx`表示是 EECS Department的课程；`18.xx` 表示的是 Mathematic  Department 课程。
  * 课程上标后缀：`上标¹` 表示春季授课，`上标²` 表示秋季授课。
  * 18.0x **A** - Alternative，适用于高中就上过微积分的学生，授课时间短。主要用于复习。
  * 18.02**2** - 数字后缀一般都是更难或者有试验，物理课程类似。
    *Additional material is included on geometry, vector fields, and linear algebra*。（看起来这个更适合计算机啊）[微积分课程说明](https://math.mit.edu/academics/undergrad/first/calculus.php)
  * x.xx **[J]** - "Joint"后缀，表示由多个部门联合授课的意思。一般是同一门课程，但是有多个不同的课号。
  
* 课程要求中的简写说明：

  * GIR - General Institute Requirements，指公共必修课。
  * HASS - Humanities, Arts, and Social Sciences，人文、艺术、社会科学。
  * CI-H - Communication-Intensive 目的在于沟通要求，可能是培养表达的课程
  * REST - Restricted Electives in Science and Technology，指的专业限选课。
  * AUS - Advanced Undergraduate Subject 高级本科科目，就是网站里列出的课程集合。
  * UR / UAR - Undergraduate (Advanced) Research，目的主要是培养沟通能力。（有研讨会的课程）

* IAP - Independent Activities Period，课程可以在 http://student.mit.edu/catalog/m6a.html 找到。

  > The Independent Activities Period (IAP) takes place for four weeks in January and is an integral part of the Institute's educational program. 
  >
  > Students, faculty, staff, and other members of the MIT community organize about 600 noncredit activities and 100 for-credit subjects. These are publicized on the IAP website, beginning in October. 
  > In addition, there are many individually arranged projects that are not officially publicized.

#### 本科教学培养计划

整个大学一共分为四个阶段，我们应该认为这是循序渐进的过程。

* **介绍性课程*Introductory Subjects***：这些课程提供了一个机会，让学生们使用EECS领域中基础性的原则，去**识别**、**建模（*formulate*）**、**解决**实际的工程问题。
  * 第一年学的主要目的是能获得基本的编程技巧、和扎实的数学基础。
  * 同时让学生们能明智的发现并选择自己喜欢的基础学科。
* **基础学科*Foundation Subjects***：旨在为学生们提供的接下来的十年内都很好的服务于（ *serve students well*）学生们的基础专业知识。
* **前沿科目*Head Subjects***：参与 3~4 个前沿科目的学习。
  * 由基础学科引出，帮助学生们在某一个领域构建更深入的认知（ *build depth*）。
  * 为未来的硕士课程、雇主们感兴趣的专业领域提供一个坚实的 *Solid Credential*。
* **高级本科科目*AUSes***：x2。旨在提供必要的基础知识，去发展自己感兴趣专业领域（一般都是最前沿的）。

根据不同的主修课程，有可能会要求参加一些 *Department LAB*，自主探究（*Independent Inquiry*）科目，或者在院内达到一定的知识广度。

以上计划，可以通过下面的 Road Map进行说明。大部分的学生都会在6个学期（不算公共学院的学期）自主安排这些学科的学习，通常每个学期会学 2-3 个科目。

![6-3-curriculum]({{ site.baseurl }}/images/2020-05-22-mit-self-teaching/6-3-curriculum.png)

图：培养计划，有必要可以自己构建一个学期学习计划表格。

#### 课程安排

我根据 6-3 和18-c 的课程网站，将课程之间的依赖关系制作成了网络，以图片的形式提供给大家。

![gir_n_introduction_subjects]({{ site.baseurl }}/images/2020-05-22-mit-self-teaching/gir_n_introduction_subjects.png)

图：公共基础必修课 + 导论课程

##### GIR 课程

18.01 微积分 I - 单变量微积分：https://ocw.mit.edu/courses/mathematics/18-01-single-variable-calculus-fall-2006/

18.02  微积分 II - 多变量微积分：https://ocw.mit.edu/courses/mathematics/18-02-multivariable-calculus-fall-2007/

8.01 物理 I - 经典力学：https://ocw.mit.edu/courses/physics/8-01sc-classical-mechanics-fall-2016/

8.02 物理 II - 电与磁：https://ocw.mit.edu/courses/physics/8-02-physics-ii-electricity-and-magnetism-spring-2007/

##### 18-c的课程

不在 6-3中出现，但是对图形学是很有必要的。

18.03 微分方程：https://ocw.mit.edu/courses/mathematics/18-03-differential-equations-spring-2010/

18.05 概率论与数理统计：**这是一门限选课，和AUSes是同级的。**
https://ocw.mit.edu/courses/mathematics/18-05-introduction-to-probability-and-statistics-spring-2014/

18.06 线性代数：https://ocw.mit.edu/courses/mathematics/18-06-linear-algebra-spring-2010/

~~看情况离散数学暂时不需要学。~~

##### [导论科目](https://ocw.mit.edu/courses/intro-programming/)

6.001 计算科学导论 - python程序设计：https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-0001-introduction-to-computer-science-and-programming-in-python-fall-2016/

* [选修，共修] 6.002 计算导论 - 思维和数据科学：https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-0002-introduction-to-computational-thinking-and-data-science-fall-2016/

EECS导论，三选一，机器人、网络通信、嵌入式系统。但是OCW网站上都统一了：

* https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-01sc-introduction-to-electrical-engineering-and-computer-science-i-spring-2011/
* https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-02-introduction-to-eecs-ii-digital-communication-systems-fall-2012/

6.042 [J] / 18.062 [J] 计算科学中的数学：https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-042j-mathematics-for-computer-science-spring-2015/



![foundation_n_head_subjects]({{ site.baseurl }}/images/2020-05-22-mit-self-teaching/foundation_n_head_subjects.png)

图：基础学科和前沿科目

##### 基础学科科目

6.004 计算结构：https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-004-computation-structures-spring-2009/

6.006 算法导论：https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-006-introduction-to-algorithms-fall-2011/

~~6.009 程序设计基础：对我来说不太重要了~~

##### 前沿科目

[二选一] 6.034 人工智能：https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-034-artificial-intelligence-fall-2010/

[二选一] 6.036 机器学习导论：https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-867-machine-learning-fall-2006/

6.031 软件构造元素：https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-005-elements-of-software-construction-fall-2008/

6.033 计算机系统工程：https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-033-computer-system-engineering-spring-2018/

[二选一] 6.046 [J] / 18.410 [J] 算法设计和分析：https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-046j-design-and-analysis-of-algorithms-spring-2015/

[二选一] 6.045 [J] / 18.400 [J] 自动装置，可计算性和复杂度：https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-045j-automata-computability-and-complexity-spring-2011/



![aus_subjects]({{ site.baseurl }}/images/2020-05-22-mit-self-teaching/aus_subjects.png)

图：本科高级科目：由于AUS只用选2个，我把大部分的都过滤掉了。由于另外两门没有找到公开课，我也从图中去掉了。

##### 本科高级科目

6.816 多核程序设计：https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-189-multicore-programming-primer-january-iap-2007/

6.837 计算机图形学：https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-837-computer-graphics-fall-2012/

~~6.815 数码和计算摄影：有说到 Ray Tracing 的PPT，但是没有公开课程
https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-837-computer-graphics-fall-2012/lecture-notes/MIT6_837F12_Lec13.pdf~~

~~6.819 高级计算机视觉：同样没有公开课程~~

### 总结

至此，我们通过整理 MIT 网站信息，完成了自学的“收集”和“整理”工作。不仅梳理了完整的课程列表，还从数学系中补充了图形学必要的学科信息（可惜依赖关系没确定），要对自己有信心鸭。

> 很有趣的是，整理过程中在google发现了一本介绍大学的书：《走進跨領域和自主學習的通識課》有人在在和我做同样的事情，就不知道他做的会不会更好了。



## 设计学习计划

### 构建学期学习计划

虽然这部分和主题关系不大，但是我认为这个小节才是自我提高最最最最最核心的部分。我们要坚决反对那些不针对学习目标的行为艺术。

![cources_table]({{ site.baseurl }}/images/2020-05-22-mit-self-teaching/cources_table.png)

图：分阶段的课程列表

所以接下来，我们要完成自学的第三部工作，为自己安排学习计划。有了这个阶段性的课程表格之后，只需要根据授课时长来规划自己的时间了。

#### MIT Unit 课时说明

这里漏掉了一个很重要的概念，MIT 使用 Units 作为课程时长的计量单位。MIT一般使用 (A-B-C)的方式来计量课程时间。

* A units = Lecture 课堂时间 
* B units = laboratory, design,  field word, 实验时间
* C units = outside preparation, 作业时间

官方给出的换算公式是： `1 Units = 14hour`， `3 Units = 1 Semester Hour`。

通常 MIT会将一个 12 Units 的课程的设置为 (5-0-7) or (4-0-8)，一门课程授课时间为 4x14~5x14 = 56~70 个小时。

* 一般来书美国一个学期有三个月左右，大概是十四个星期。`1 Semester = 3 months = 12~14weeks`
* 如果按照一个学期学完一门 12units 的课程，换算过来：56 hour ÷14 weeks = 4 h/w。
* 正好是对应上 `1 weak = 4h-0h-8h = 12 hours`。大概每周需要4小时听课时间，8小时写作业时间。
* **写作业的时间是听课的1.5~2倍时长。**

*p.s.一般来说，MIT的学生每一年只会选 3~4 门课。业余时间一个星期 12小时遭不住啊。*

有了这个时间概念，我们就可以遵循 MIT 的建议，根据学期列课程表格。到这一步位置，您应该和我一样，拥有了一份以年为单位的计划表。题外话，由于四个阶段的目标非常明确，个人觉得非常适合 OKR 的方式给自己构建计划。

<table style="text-align:center;">
    <tr>
        <th>学年</th>
        <th>目标</th>
        <th>关键指标</th>
        <th colspan="3">课程列表</th>
    </tr>
    <tr id="year1">
        <td>1</td>
        <td>数学基础</td>
        <td>熟悉微分、积分应用</td>
        <td style="background-color:#FFF2CC;">18.01</td>
        <td style="background-color:#FFF2CC;">8.01</td>
        <td></td>
    </tr>
    <tr id="year2">
        <td>2</td>
        <td>数学基础</td>
        <td>熟练应用积分解决物理问题</td>
        <td style="background-color:#FFF2CC;">18.02</td>
        <td style="background-color:#FFF2CC;">8.02</td>
        <td rowspan="2" style="background-color:#FCE4D6;">18.03</td>
    </tr>
    <tr id="year3">
        <td>3</td>
        <td>数学基础</td>
        <td>熟练使用数学工具解决3D问题</td>
        <td style="background-color:#FCE4D6;">18.06</td>
        <td style="background-color:#E2EFDA;">6.006</td>
    </tr>
    <tr id="year4">
        <td>4</td>
        <td>图形学入门</td>
        <td>掌握图形领域的核心问题</td>
        <td style="background-color:#DDEBF7;">18.05</td>
        <td rowspan="2" style="background-color:#DDEBF7;">6.837</td>
        <td></td>
    </tr>
    <tr id="year5">
        <td>5</td>
        <td>图形学入门</td>
        <td>熟悉光照方程和背后的物理基础</td>
        <td style="background-color:#F0F0F0;">其他材料</td>
        <td></td>
    </tr>
</table>
![my_study_schedule]({{ site.baseurl }}/images/2020-05-22-mit-self-teaching/my_study_schedule.png)

图：我猜 GitHub Pages 不支持 HTML table，所以加了个图。（要脸，所以这个表格是随便写的。）

“计划”步骤非常最重要，它能让我们在养成习惯、回顾成果的时候有明确的对照，帮助有非常大。

### 如何使用网站材料

这部分我实践了好几年，但是没有特别系统的方法和步骤。权当探讨和大家分享大概的步骤：

1. 阅读视频，了解基本的概念，当做是函授教程。上完课后，认真记录笔记。
2. 阅读网站的本章的学习目标，买实体书并进行章节学习。
3. 写作业 + 回顾网站的关键问题解答笔记。
4. 学习完成总结笔记，构建知识网络。

最近发现**实体书**是非常重要的学习资料，能随时随地的阅读；而且可以补齐视频里错过的重点理论知识。

每个人都有自己的习惯吧，各位小伙伴如果有更优秀的实践方法，非常希望您能教教我。

> “路虽弥,不行不至；事虽小,不做不成”  --《荀子·修身》
>
> 道阻且长，行则将至。与君共勉。



## 附录：旧版本的网站截图

由于篇幅有限，课程信息没有记录太完整，所以我单独整理了一个 excel 表格，保存在版本库里。

在此主要记录 MIT 旧版网站删除的一些信息，我觉得有A-B对比会有更深入的见解，所以都保存下来了。



![roadmap_6_3_2016]({{ site.baseurl }}/images/2020-05-22-mit-self-teaching/roadmap_6_3_2016.png)

图：2016年的road map



![roadmap_6_3_old]({{ site.baseurl }}/images/2020-05-22-mit-self-teaching/roadmap_6_3_old.png)

图：很古老的 road map