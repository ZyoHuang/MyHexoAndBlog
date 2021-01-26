---
title: 目标导向的AI系统（GOAP）技术分享
date:
updated:
tags:
categories:
keywords:
top_img:
cover:
aplayer:
---
<meta name="referrer" content="no-referrer" />

如果让大家设计一个AI框架的话，相信很多人都会选择FSM或者行为树之类的插件或框架。其实这两种方案都可以归类于我们提前写死AI逻辑然后运行时直接或间接遍历整个逻辑图/树来表现AI这一大类里，也正是因为他们这种特性，我们会被越来越复杂的AI逻辑折磨的痛不欲生，往往加入一个新的AI功能，整个AI逻辑都要发生巨大的变动。而GOAP则是我们只需要提前做好各个Action和Goal，不用考虑太多状态的切换，他会在**运行时根据Goal来自动给出AI方案**，是不是听起来很神奇？那接下来我来带领大家一点点揭开GOAP的神秘面纱。


## GOAP的思想

GOAP思想受STRIPS启发，STRIPS是由斯坦福大学于1970年开发的，名称是斯坦福研究所问题解决者（Stanford Research Problem Solver）的首字母缩写。 一个Action只有在其所有前提条件都得到满足的情况下才能执行，并且每个动作都会以某种方式改变世界的状态。

**GOAP的功能组件大致分为Action，Goal，Memory，Sensor，他们各司其职，构成整个GOAP框架。**

- Action：它由前提条件（PreCondition）和效果（Effect）组成，一般都是键值对，要执行一个Action，需要先检查前提条件是否满足，然后再把效果内容加入到Memory中（可选）。例如：![image-20200730093406358](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200730093406358.png)

- Goal：AI的目标，也是包含一个或多个键值对，会以此为基准使用A\*算法展开路径搜索。例如：![image-20200730093550000](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200730093550000.png)

- Memory：AI的记忆，更新维护AI所认知的世界状态，偏静态。例如：{'hasMoney': true}
- Sensor：感知者，其本质和Memory一样，但是偏动态，之所以需要他是因为我们很多时候需要做一下逻辑处理才会把世界状态写入Memory，例如，AI在血量小于30%时会寻找掩护物，要处理这个过程，我们需要一个Sensor来订阅AI的血量改变状态，每次改变就判断是否小于30%，如果是，就把Memory中”isInDanger”设置为true。

## GOAP的原理

前面我们提到GOAP是根据Goal来规划AI路径的，那么它的具体原理是什么样子的呢？我们这里以[ReGoap开源库](https://github.com/luxkun/ReGoap)为例。

![image-20200730093651093](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200730093651093.png)

这只是粗略的架构版本，其本身的细节和内容要比这个多得多。但是大体流程是这样子的。

所以我们可以看到ReGoap其实是把整个AI系统分成了两大部分。

**第一部分是根据目标来规划路径部分，这部分使用了A\*算法，从目标点开始往外扩张，因为这部分是纯逻辑计算，所以可以做成多线程的，每个线程都参与到路径的规划中，分摊主线程计算压力，然后把路径规划结果（包含很多Action的列表）通过回调函数传回到主线程，我们就可以正式去执行Action的逻辑了。**

**第二部分就是上一段提到的执行Action逻辑部分，这部分只能在Unity主线程进行，因为有些逻辑可能会需要用到Unity的API。一个Action执行后，如果成功就执行我们得到的路径里面的下一个Action，如果失败，说明这个路径实际操作不可行（这会与游戏中实时的一些因素有关，这也是没有办法的，因为我们不可能把所有可能的因素都放入到Memory中，那样的话可操作性和性能上都不会让人满意，这一点在ReGoap的FSMSample的示例里有展现），就重新选择Goal，规划路径。如果一个Action列表全部执行成功，说明Goal达成，可以选择重新选择Goal，规划路径，也可以选择进入待机状态。**

## GOAP的使用场景

所有需要用到复杂AI的地方都可以使用，为什么非要是复杂AI呢，因为简单AI用FSM或行为树足以干净利落的解决问题，就没必要舍近求远了，那么这个复杂和简单用什么来界定呢，我在这里说一下我的标准，如果一个FSM的连线或者切换关系已经让人感觉一团糟，一个行为树的树状图非常庞大（100起），这个时候，你就可以考虑使用GOAP来化解难题了。

## GOAP的示例

这里以ReGoap自带的单元测试用例为基础，再进行拓展和修改，来讲解GOAP具体该怎么使用及其运行规则。

![image-20200730093736504](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200730093736504.png)

**那么他所描述的世界是这样的，每个条目前的字母为代号，ActionGroup（浅绿色）中的括号值代表Action的权值（在A\*规划的时候青睐权值较小的Action，因为这样消耗最小），GoalsGroup（浅粉色）中的括号值代表Goal的优先级，值越大，优先级越高，会先被纳入规划目标中。**

![image-20200730093753344](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200730093753344.png)

那么，我们现在开始进行规划，现在有两个目标，而GoalZ的优先级大于GoalY，所以会先把GoalZ纳入规划目标，但是我们找不到“贪婪的”这一状态，所以这个目标规划失败了，然后去尝试规划GoalY，它的规划结果如下

![image-20200730093809837](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200730093809837.png)

**即：ActionA->ActionC->ActionB->ActionD**

**这似乎与我们所想的ActionA->ActionB->ActionC->ActionD结果不同，这是因为ReGoap中使用的A\*是从终点->起点，而不是起点->终点的，这就会导致它只可以保证综合结果的Cost一定是所有规划里最小的，但在单次规划中并不会确定性采用权值较小的Action为下一结点，这样做是有考虑的，因为如果是从起点到终点进行的规划，就会多出非常多的可能性，更多的可能性意味着更多的计算，这对于要求低延迟的游戏造成的后果是不可接受的。**

接下来我来论证上面的高亮观点，新增一个ActionF

![image-20200730093834323](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200730093834323.png)

那么得到的结果会是

![image-20200730093846409](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200730093846409.png)

即：ActionF->ActionB->ActionC->ActionD，我们发现，不仅结果改变了，Action的执行顺序也改变了。但是它确实是选择了消耗最小的路径！

**那么如果我们想强制AI进行ActionA->ActionB该如何操作呢？我们可以做以下修改**

![image-20200730093910302](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200730093910302.png)

结果就是

![image-20200730094024192](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200730094024192.png)

**即：ActionA->ActionB->ActionC->ActionD**

Action相关的到这里就差不多了，接下来我们来看Memory，还记得我们有两个目标吗，现在该临幸另一个了，在此之前，先把现在这个Goal安顿好，我们把这个Goal所对应，Action列表中的状态写入到Memory中，现在的Memory中的内容应该是“没有石材”，“没有木材”，“没有原石”，“没有原木”，“有房子”，之前是因为我们记忆中没有“贪婪的”这一状态，所以失败了，那么我们在Memory新增上这个状态，就变成了“没有石材”，“没有木材”，“没有原石”，“没有原木”，“有房子”，“贪婪的”

![image-20200730094029730](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200730094029730.png)

我们再次开始进行目标规划

![image-20200730094035863](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200730094035863.png)

**即：ActionC**

这个例子到这里能讲的东西差不多都讲完了，相信大家都对GOAP有一定的基础认知了，不过ReGoap所提供的功能远不止这些，还包括Goal规划打断，动态Action等。

当然了，可能有观众会说，这个例子用FSM明明更好做啊，确实，在这种情形下GOAP确实没什么优势，但是问出这个问题的观众可以去看[FEAR基于GOAP的AI系统GDC分享（中英双语）](https://link.zhihu.com/?target=https%3A//www.lfzxb.top/gdc-sharing-of-ai-system-based-on-goap-in-fear-simple-cn/)中的这一张图相关内容，相信你会找到自己想要的答案。

![image-20200730094041480](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200730094041480.png)


## 对比GOAP和FSM/行为树优劣

其实前面的内容已经多多少少沾点这部分的内容了，这里做一下总结和补充。

**先说缺点**

- GOAP学习成本偏高
- GOAP默认不保证Action顺序，如果我们想要强制指定顺序，需要额外加State（就像上面那个例子一样），但是一般而言我们的AI也不会去在意这种无关紧要的顺序，因为重要的顺序我们已经规范好了。
- GOAP毕竟使用了运行时规划，会对游戏性能产生一定的影响，但是这种目标导向的A\*规划在AI复杂的情况下，性能并不一定比FSM/行为树弱。

**再说优点**

- 解耦目标和行为，设计时可以更加专注的设计AI的行为不用过多考虑目标，新增，减少Action非常方便，不会像FSM/行为树那样具有那么强的入侵性和破坏性，提高了开发效率
- 动态规划的能力，这会使我们的AI看起来更加“智能”
- 配置方便，易于理解，完全可以让策划使用EXCEL接管AI开发工作



## 总结

GOAP据我了解是在2003年提出的概念，但是直到现在用的人也不是很多

估计应该是对于程序员来说的学习成本和上手难度导致的，但是其本身的理念是优秀的，无论是程序员的编码还是和策划的配合工作上，都是一个创新和改革，完全可以尝试在我们自己游戏中应用，我后面也准备结合一下Unity的JobSystem做一下路径规划耗时优化，感兴趣的可以关注：



- [ReGoap 开源库](https://github.com/luxkun/ReGoap)
- [ReGoap 开源库中文文档](https://www.lfzxb.top/goal-oriented-action-planning-chinese-document/)
- [GOAP 资源汇总](http://alumni.media.mit.edu/~jorkin/goap.html)
- [MKGGoap 开源库（多线程，JobSystem，路径规划算法优化）](https://gitee.com/NKG_admin/NKGGOAP)
