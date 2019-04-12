# LittleBee
[![996.icu](https://img.shields.io/badge/link-996.icu-red.svg)](https://996.icu)
[![LICENSE](https://img.shields.io/badge/license-Anti%20996-blue.svg)](https://github.com/996icu/996.ICU/blob/master/LICENSE)
## 0x00. 引言
ECS是Entity-Component-System（实体-组件-系统） 的缩写，是一种非常好用的框架思想，可以提高代码复用率，游戏逻辑开发中使用这种组合优于继承的方式可以很大程度上简化复杂度，而且在性能上也是有很大提升的。Entiy是一个包含唯一ID的容器对象，在Entity内部可以绑定很多Component，每个Component只负责存储数据，正是因为Component的功能单一性，它可以很方便的做数据快照。System内部负责操作Component数据，从而最终作用于Entity。

帧同步的应用场景很多，（帧同步与状态同步的区别，感谢知乎作者云影）从格斗游戏，Rts，到现在的几乎所有Moba类游戏，他们各自的实现方式细节各不相同，帧同步对不同类型的游戏也有各种变种。列举一下例子，如格斗游戏，由于参与角色不多，玩家对角色操作反馈及时性要求很高，因此这种帧同步需要有预测能力，在预测对方玩家行为之后还需要有回退矫正的能力。有预测能力意味着操作反馈会很流畅，也是由于角色不多，出现预测错误的几率也不多回退矫正造成的视觉影响也不会很大，因此格斗游戏的帧同步就要采用这种方式定制。再比如魔兽争霸游戏，这种早期局域网环境下的对局方式，网络的环境比较好，各个客户端采用的是锁帧方式的帧同步，如果其中一个客户端出现小延迟情况，客户端自己会迅速根据关键帧标记跟上服务器的步伐，如果出现大延迟甚至断线，那么所有的客户端需要一起暂停等待，因此这种方式对于多控制单位的游戏类型很好用也很简单。

## 0x01.开始结合 
准备将帧同步与ECS合起来工作，帧同步逻辑实现的第一步是要将逻辑和显示彻底的分离，可以把很多逻辑放到线程里去，这点也是ECS可以带来的便利。
首先需要实现一个模拟器，这部分的代码功能主要是实现一个stopwatch的机制，在线程中使用stopwatch记录时间流逝，在设定逻辑帧帧率参数一致下同时启动模拟器保证在经过相同的一段时间内所有的模拟器执行的帧数一样。模拟器还需要有添加行为的功能，各种行为以互相排列组合的方式共同协助模拟器完成整个模拟过程。
[ECS方式实现帧同步]
#### 逻辑帧记录：主要负责记录当前的逻辑帧编号；
#### 网络数据回滚：负责收取服务器广播的带帧编号的网络数据，这部分也是帧同步的核心，收到的逻辑帧数据需要和本地的数据进行回滚并向后继续演算，并且记录下操作以便播放战报；
#### 实体操作：在这里会是所有ECS中System的执行入口，如MoveSystem等；
#### 用户输入：完成监听并记录用户操作，用于最后一步的请求同步；
#### 数据备份：对当前帧的所有数据进行快照保存，以便用户回滚；
#### 请求同步：将操作发送给服务器；
整个顺序过程会在一个逻辑帧内执行，因为上述所有逻辑都放置与线程中执行，所以在显示方面和这部分逻辑是没有调用关系的，因此需要在主线程中对于当前的数据做出响应，这就是一个数据驱动显示的过程。这样做的好处有很多，首先就是逻辑显示分离后服务器也可以执行逻辑代码进行验证，其次线程里执行的逻辑运算功能可以分担主线程压力。

上述的模块还可以继续扩展，比如可以在用户输入模块后继续加上AI输入模块，在实体操作模块中添加子系统，子系统以ECS中的System形式出现，负责根据需求改变数据。

关于ECS的实现可以有很多种，具体情况因地制宜，当然也可以自己DIY一下。我在这里把ECS扩展成ECSR，多出来的R表示Render，Render会根据Component数据内容用于主线程画面的表现。这边为什么要把逻辑帧的执行放在线程里，因为后续需要加入物理检测用来提升运行性能和战报完整复盘与检测。
## 0x02. 扩展一下
上述完成了对帧同步模拟过程，还有一项帧同步特有的功能就是战报回放，这是状态同步开发下无论如何也完成不了的。战报回放的replay文件可以以不同速率完整的模拟出整个帧同步过程，这依赖于逻辑帧命令的记录和保存。同样是用模拟器去完成只是在模拟器中添加不同的模块。
#### 逻辑帧记录：这里和上述的稍作改变，作用相似，也是记录逻辑帧编号；
#### 实体操作：同上
#### 回放输入：根据回放的replay文件，读取逻辑帧命令；

## 0x03. 后续
关于帧同步的网络通信方面的选择，最好还是使用UDP方式，采用发送冗余信息，外加丢帧请求的方式。我这里为了简单实现同步的过程采用的是TCP方式。后续的修改应该是往UDP方式改进。
另外帧同步中的数学运算官方提供了Unity.Mathematics数学库。
