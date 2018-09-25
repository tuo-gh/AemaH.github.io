---
layout:     post                    # 使用的布局（不需要改）
title:      Reinforcement Learning               # 标题 
subtitle:   对于强化学习的一些总结 #副标题
date:       2018-09-21              # 时间
author:     ERAF                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 强化学习
---
> 照例开篇的碎碎念，好吧，也知道自己之前缺了多久没有更新，其实我真的也在写真的(认真脸) ，起码前一段时间我就从：Model-based和TD 还有IRL都下笔写了的「这不叫挖坑...」，后来发生一些事情 emmmm 于是也就没写完(长叹气) ；
> 算了 无论如何先从之前 已经总结好的一些当做一个新的开始吧（握拳）

这一次所坐的主题是强化学习中一些基本问题，包括基础的：model-based和model-free的概念、off-policy和on-policy的概念、基于值函数和基于策略梯度的、RL和SL的一些碎碎念（慎读）、RL中的分类、RL过程中predict和control的概念；

更多的还是一些想法，缺少了相关概念性的描述 之后有时间再添加吧；

## off-policy和on-policy

我们都知道这两者的分辨很简单：产生数据的策略和进行评估的策略是否一样就是了；最鲜明的区别就是SARSA和qlearning， 前者基于某一个策略如ε-greedy进行选取action，后面需要值函数进行更新的时候 也还是ε-greedy策略。后者则不然 选取action的策略是基于ε-greedy，更新值函数的时候依旧是选取max的那个；

如果深入了解的话，可以从强化学习的目的说起。毕竟强化学习的目的是为了获取最优策略，也就是一种确定性的「状态-最优action」这样一个state和action的分布对应的映射，也就是target policy； 而这儿两者对于这个target策略的获取方式的思路有所不同。

on-policy就是有针对性探寻这样一个target policy，就像SARSA一样，在这个过程中一边基于一定的要求选择action 一边对应的来对于Q值函数进行更新； 因而每次对于值函数的更新 直白的是基于交互序列进行的。这样这个过程中会带来局部最优的问题，也就是只利用当前最优选择 学不到整体最优，再放大了说 就是探索和利用的矛盾；因而ε-greedy的设计在一定程度上解决了这个问题； 

而off-policy的思路不同，它并不是直接的就是学习得到target policy，而是先基于某分布下的大量行动数据 目的在于探索，参考qlearning里面进行q值更新时候的max 选取的就是其中的最优选择， 也就是说并非按照交互序列来直接进行学习，通过选择来自其他策略的交互序列的一部分来替代自身的一部分，考虑到了子部分的最优性，进而在前面累计的最优化结果的基础上不断更新与优化。

### 好坏的话

on-policy优点是直接了当，速度快，劣势是不一定找到最优策略。

off-policy劣势是曲折，收敛慢，但优势是更为强大和通用 保证了探索性。其强大是因为它确保了数据全面性，所有行为都能覆盖。「这里所说的就是在选取$s_{next}$对应action的时候需要选取max的一个 那么需要全部action的q值 于是覆盖了全部的行为，相比之下on-policy只是单纯的依照策略选取了某个action；」

### 关于DQN里面的experience replay方法和A3C的asynchronous

显而易见的 前者的经验重放就是一种标准的off-policy的方法，毕竟在DQN里面使用的时候是在对估计Q(s,a)，estimate_Q(s, a) = r + lamda * maxQ(s2, a) ，如果按照on-policy理解的话 这个时候的应该是estimate_Q(s, a) = r + lamda * Q(s2, a2)  也就是说这个时候应该保存当时候的网络模型来进行输出对应的Q值情况，进而进行基于相应的值函数 在基于相同的如ε-greedy来选择情况；现在经验重播中只保存了$<s,a,r,s_{next}>$ 谈何on-policy呢。

同时上面也说了是直接保存的交互数据进行训练，这样观察数据往往波动很大且前后sample相互关联「机器学习也要求样本彼此独立同分布」 经验重播的方法很不适合on-policy；

但多线程的synchronous不一样；因为是多个agent在多个环境实例中并行且异步的执行和学习。 数据就不存在上面所说的数据关联性的问题。多个并行的actor可以有助于exploration。在不同线程上使用不同的探索策略，使得经验数据在时间上的相关性很小。这样不需要DQN中的experience replay也可以起到稳定学习过程的作用，意味着学习过程可以是on-policy的。 

## model-free和model-based

简单的说就是转移概率是否知道，再具体一点就是再知道状态和执行动作后 下一个时刻可能导致的后果能不能推断出来。从这种角度回头看model-free 就是一步步与未知的环境交互来试图学习最优的策略，经过多次迭代后 得到整个环境中最优的策略。

从这种来看，基于模型的方法可以用动态规划的思想进行；具体的来说 该问题可以分为多个优化子问题 而且子问题可以重复利用和存储。方法包括策略迭代和值迭代，前者的包括策略评估和策略改善两个步骤：一步步的来对于每个状态的值函数进行更新，进而获取最优策略，在反复迭代； 后者类似于只是前者的一轮操作：只更新了状态值函数一次，就进行最优策略的表示选择「只评估了一次就进行策略改善」，但这里不同的是引入一种想法：因为有最优策略的存在 所有策略的选择最终就是单一的 最优策略的表示肯定不会比其他的差，因而 只是单纯的找到那个贪婪的 值最大的；

这里之所以说是对于全部状态进行更新，就是考虑到一个概念就是：**时间不变性**：指的是随着MDP的过程的进行，最终不同时刻下的相同状态总会有着相同的价值；

而实际问题中 更多的依旧是model-free 「某一个状态的之后状态的转移概率未知 那么就无法根据贝尔曼方程来直白的计算，于是我们就需要先知道当前这条markov链」于是对于这一类markov链的状态值函数的求解 也就是一类多步学习预测问题，常见的方法包括蒙特卡洛方法 时域差值学习； 

首先是对于蒙特卡洛的首先进行介绍，简单的说就是基于经验（experience）进行学习，这个经验包括样本序列的状态（state）、动作（action）和奖励（reward）。得到若干样本的经验后，通过**平均所有样本的回报（return）**来解决强化学习的任务。

但显然能看出蒙特卡洛的方法，尽管可以作为一种类似监督学习的方法「基于历史数据 进行预测」可以解决多步预测的问题，但是有着效率低下的特点；而时域差分的方法 利用连续两个时刻预测值的差值来更新模型；

如果继续往下看的话，包括Qlearning、SARSA等控制学习算法的基础 也都是在这里，进而我们常见的强化学习中所分类得到的表格式和值函数逼近式的，也正是因为这里的TD算法的分类而导致的；进一步的还有对于单步预测和多步预测还有着：TD(0)、TD(λ)；

>   顺带说一句：根据观测数据的时间特性 预测可以分为单步预测和多步预测「基于历史数据预测现在和未来的区别」；关于监督学习和时间差分的区别，**传统**监督学习是根据预测值和实际观测值的误差来修正预测模型，而TD是根据时间上连续两次预测之间的差值来修正预测误差；具体的来说 前者需要得到全部观测数据后 才能通过计算预测误差来修正预测模型，后者 只需要某两个时刻的预测值和局部观测数据来修正预测模型；所以可以实现在线学习 减少存储量和计算量 有着更高效的学习效率；同时 也可以看出 监督学习只能实现单步学习预测 也就是只能基于当前信息来对于当前时刻的输出进行预测，而TD可以实习多步预测；
>
>   时域差分算法 往往被看组学习控制算法 如sarsa和qlearning的一部分 就像多步预测被看做为多步控制的一部分一样；

## 蒙特卡洛、TD、动态规划

有关确定值函数描述的几种方法：动态规划的值迭代、蒙特卡洛方法、TD方法；蒙特卡洛上面也说一些，其对于值函数的定义还是值函数最原始的定义，于是需要知道全部的回报「基于最终状态」进行积累才确定；而无论是 值迭代 还是TD方法 中计算某状态「当s为$s_{t}$」值函数的时候，从其定义公式$E_{\pi}[R_{t+1}+\gamma v_{\pi}(s_{t+1})]$ 显然知道 计算这个值函数 需要知道所有后续状态以确定当前状态的值函数，但不同之处在于 动态规划的方法的依托是model-based，也就是说能够自我推导后续全部状态；而TD借助实验才能得到后续状态 ，所以 这里我们需要设置多次episode 以遍历尽量多的状态；

## 基于值函数的方法和基于策略的：

关于值函数的方法，总结性的来说就是：值函数近似的方法是通过描述构建起来一个评估函数：值函数「评估策略的好坏的」，进而优化这个值函数 使之达到最优，当其最优的时候，作为最能描述策略好坏的值函数，那么此时 采用**贪婪策略「确定性策略」或者ε-greedy「随机策略」** 在某状态 选取对应的action，也就是当前状态所选的最优动作， 整体来说 也就是最优策略了；

于是也能看到，对于值函数方法中，努力优化的是值函数，以此作为评判标准来评价其对应的策略的好坏；但不得不说 作为值函数方法本身 作为一个评价函数需要有着确定化的输入「状态」和输出「动作 或者 Q」，当面对诸如 连续动作的时候并不能有着很好的表现；

于是引入了直接的策略搜索的方法；我们先说一下关于两者实现的区别：值函数的方法提过很多遍了，是需要先构建好值函数 然后优化好值函数，才能得到策略；而策略搜索的方法不同 则是使用一个参数化的线性或者非线性函数用以描述策略 然后寻找最优的策略，是强化学习的目标：累计回报的期望 最大；

于是 策略搜索 顾名思义的也正是在策略空间直接进行搜索，不再借助值函数那样的间接完成对于最优策略的确定；于是也带来了收敛性快这样的好处；方法包括策略梯度 DDPG等

策略梯度的方法收敛性快，可以有效地处理连续动作集的问题「值函数方法无法确定一个对应max Q的action」，但同样的容易收敛到局部最小值、方差较大。

从似然函数的角度简单推导一下策略梯度，也就是借助诸如SGD需要求解的梯度；

>   上面 我们说过无论是基于值函数还是基于策略搜索的方法，本质上的目的也都是强化学习的目的：让累计折扣回报最大化；对于一个马尔科夫过程 所产生的一组状态-动作序列 我们用$\tau $表示$s_0,a_0,s_1,a_1...s_H,a_H$
>
>   那么对于这么一组状态-动作对 或者说这一条轨迹的回报我们用 $R(\tau)=\sum^{H}_{t=0}R(s_t,a_t)$，而该轨迹出现的概率我们使用$P(\tau;\theta)$表示，那么显而易见的从强化学习的目标：最大化回报 这个目的出发，可以写作 $max_{\theta}U(\theta)=max_{\theta}E(\sum^{H}_{t=0}R(s_t,a_t))=max_{\theta}\sum_{\tau}P(\tau;\theta)R(\tau)$ 「从回报期望的形式 到写作与概率相乘的形式，之前似然函数的部分说过」
>
>   进而为了优化这个目标函数，常见的如GD 找到那个最快下降的方向，于是对其求导有：
>
>   $\nabla _{\theta } U( \theta ) =\nabla _{\theta }\sum\limits _{\tau } P( \tau ;\theta ) R( \tau ) $
>
>   $=\sum\limits _{\tau } \nabla _{\theta } P( \tau ;\theta ) R( \tau ) $
>
>   $=\sum\limits _{\tau }\frac{P( \tau ;\theta )}{P( \tau ;\theta )} \nabla _{\theta } P( \tau ;\theta ) R( \tau ) $
>
>   $=\sum\limits _{\tau } P( \tau ;\theta )\frac{\nabla _{\theta } P( \tau ;\theta ) R( \tau )}{P( \tau ;\theta )} $
>
>    $=\sum\limits _{\tau } P( \tau ;\theta ) R( \tau ) \nabla _{\theta } logP( \tau ;\theta ) $
>
>   于是 梯度的计算转换为求解$ R( \tau ) \nabla _{\theta } logP( \tau ;\theta ) $的期望，此时可以利用蒙特卡洛法近似「经验平均」估算，即根据当前策略π采样得到m条轨迹 ，利用m条轨迹的经验平均逼近策略梯度 于是有：
>
>   $$\nabla _{\theta } U( \theta ) \approx \frac{1}{m}\sum\limits ^{m}_{i=0} R( \tau ) \nabla _{\theta } logP( \tau ;\theta ) $$

如此 我们就得到策略梯度的推导结果；如果直观的理解上面的公式的话，$\nabla _{\theta } logP( \tau ;\theta )$ 就是轨迹随着参数θ变化最陡的方向，而 $R( \tau )$ 代指了步长 影响了某轨迹出现的概率；

进一步的 由似然函数 损失函数中的log部分可以推导为只包含动作策略的 $log\pi_{\theta}(a;s)$ 「转移概率无θ可删去；具体步骤可以参考RL :introduction」，因包含策略的表示，根据需要有着多种方式「具体参考RL:introduction」本文我们仅通过实例来进行说明 所构建出来的损失函数；

### 基于值函数的方法和基于策略的交叉点的AC方法

两者的交叉也就是actor-critic。也就说是 以值函数为中心 结合上 以策略梯度为中心；

actor是基于策略梯度的方法进行选取动作，而critic是基于值函数的方法来评价它，两者协作完成；

于是我们可用先从策略梯度的角度来理解：

-   普通的策略梯度中loss function的表示是 $logP(\tau;\theta )R(\tau)$ 前者代指的是方向,进而基于θ求偏导的话 毋庸置疑也就是让轨迹τ的概率变化最快的方向，或快速增加或快速减少，只是取决于正负号；后者的是$R(\tau)$ 作为一个标量 类似于前者的增幅 当其为正的时候，当这个值越大的时候 轨迹τ出现的概率$P(\tau;\theta)$ 在参数更新之后会越大「建立的函数是两者相乘 但更新的参数只是在前者这个描述出现概率的变量中出现」反之则越来越小。所以，策略梯度的方法会增大高回报轨迹对应的出现概率 而会降低低回报轨迹对应的出现概率；
-   从这个角度再来理解actor-critic，轨迹的回报$R(\tau)$ 可看做一个critic；用于评价参数θ更新后 该轨迹出现的概率是该增大还是减少 以及对应的幅度；对应的可以表示成：

![](https://ws1.sinaimg.cn/large/005A8OOUgy1fu4zxc8pctj30h90ev0vz.jpg)
    

```
>   对于他们的评价如下：
>
>   1--3：直接应用轨迹的回报累积回报，由此计算出来的策略梯度不存在偏差，但是由于需要累积多步的回报，因此方差会很大。
>
>   4—6: 利用动作值函数，优势函数和TD偏差代替累积回报，其优点是方差小，但是这三种方法中都用到了逼近方法，因此计算出来的策略梯度都存在偏差。这三种方法以牺牲偏差来换取小的方差。当$\varPsi_t$取4—6时，为经典的AC方法。
```

​    

于是一个广义的AC框架 就是：前面的 $\pi_{\theta}(a|s)$ 是actor作为策略函数；
后面的 $\varPsi _t$ 是critic 评价函数 「换句话说 critic使用的就是各种策略评估方法」

### Actor - Critic的优点

（总的来说就是结合两种方式优点）：

-   相比以值函数为中心的算法，Actor - Critic应用了策略梯度的做法，这能让它在连续动作或者高维动作空间中选取合适的动作, 而 Q-learning 做这件事会很困难甚至瘫痪。

-   相比单纯策略梯度，Actor - Critic应用了Q-learning或其他策略评估的做法，使得Actor Critic能进行单步更新而不是回合更新，比单纯的Policy Gradient的效率要高。

  ​    

因为AC的方法，是在之前的策略梯度的方法之上 将其中的用于描述轨迹累计reward的项 转化为了行为价值函数 $Q_{\omega}(s,a)$  同时这里的状态值函数，接下来是一个简单实现的算法 行为值函数采用线性表达 于是更更新的过程中：critic通过近似TD(0)来更新ω，actor借助策略梯度更新θ，整体描述有：

![](https://ws1.sinaimg.cn/large/005A8OOUly1fvcxnjry2qj30gf09xdga.jpg)

可以看到 正是个实时的在线学习；针对每一步 一方面基于策略选取动作a和a‘，这时候的策略选取不再需要ε-greedy来获得，而是直接借助参数θ得到；然后得到TD-error，先类似于策略梯度中那样更新actor的参数θ；再更新critic中的参数ω用于生成值函数；

### Actor - Critic的总结

「回头看整个AC架构，就是在策略梯度更新的基础上 将其中的的元被用于描述整个轨迹的累计reward的$R(\tau)$ 改写成动作值函数；前者的策略梯度本身就是对于策略本身的描述 于是借助于它 我们能基于状态得到action；然后改写成的动作值函数 可以得到对于该state-action的评价值；进而回头类似于策略梯度的更新 更新这个策略」

再回头对比想想看和之前两者的区别：

1.  和策略梯度的区别，很明显 关于策略梯度的算法表述里面 并没有涉及关于critic部分或者具体点说就是值函数更新的部分；参考RL:an introduction 里面所描述的那样P292中所描述的那样： 策略梯度方法里 状态值函数更多的是作为一种基准而非是critic；也就是说只是作为一种基准来判断哪个状态需要被更新「for state-function」**而事实上 为了实现Policy Gradient，不管我们是计算Q，还是V，都需要一个对应的网络，这就是Critic。换句话讲，我们只有在使用Policy Gradient时完全不使用Q，仅使用reward真实值来评价，才叫做Policy Gradient，要不然Policy Gradient就需要有Q网络或者V网络，就是Actor Critic。** **「对于其中采用什么值来当做$R(\tau)$ 参考这来理解https://zhuanlan.zhihu.com/p/26882898」**
2.  和值函数的方法 差别就更大了；首先对于策略的描述 就不是值函数方法里面借助 $argmax_a Q(s,a)$ 而是使用的策略梯度的方法；当然使用对于该策略的好坏的评价 但也是一样的值函数；

然后对于一整个actor-critic的训练过程如下：

![](https://ws1.sinaimg.cn/large/005A8OOUgy1fu4zxspcrzj309w09st8t.jpg)

如图 基于策略梯度的actor基于概率来对于某状态来选取动作action；而critic基于actor的行为判别行为的得分；actor进而基于该评价值来计算出来一个td error修改选择行为的概率「换句话说就是：actor的策略梯度的方法生成得到梯度的方向「也就说之前的$logP(\tau;\theta )$」，然后进行沿着方向进行梯度的增减；我们需要一个值来判断这一个增减的方向是否正确 于是需要critic来计算出来td error」，同时基于选取的action计算TD error来更新critic；

具体到网络里面：Actor和Critic各为一个网络，Actor输入是状态输出的是动作，loss就是$log_{prob}\times td_{error}$,(和策略梯度相对应，注意到这里的loss和Policy Gradient中的差不多，只是vt换成td_error，引导奖励值vt换了来源（Critic给的）而已)，Critic输入的是状态输出的是Q值，loss是square((r+gamma*Q_next) - Q_eval)也就是square(td_error)，也就是说这里更新critic对应Q-learning是一样的均方误差。 

网络的交互也和上图一致：agent每次状态1从actor中得到一个动作a1 和env类交互得到s2和即时奖励r。然后把s1s2 r输入critic网络，更新其中的参数ω并计算得到td_error；然后把a1，s1，td_error输入到actor网络更新其中的参数θ；「TD_error信号同时指导actor网络critic网络的更新 」



## 强化学习和监督学习

强化学习的起源，那就需要追溯到动物学习心理学有关“试错”学习，即 **强调在与环境中的交互进行学习**，并基于环境对于不同行为的评价反馈信号用于改变自身的动作选取策略来达到学习的目标；这里就出现了一个关键词：**交互性**；

没错，对于RL(reinforcement learning)来说，毋庸置疑的一点就是：RL的重点在于交互；那么交互的双方呢？参考先前的动物学习心理学里面，我们可以用agent和environment两类来代指交互双方；实际上，在实际编程过程中， 我们往往需要设置两部分：agent和environment；前者负责各类RL算法的内容：基于策略选取action、优化更新策略等，后者需要根据agent选取出来的action 进行做出反应「给予回报和环境状态变化情况等」；这里 可以引出有关强化学习的第二个关键词**机器学习**；

强化学习是机器学习的一种，但某种意义上又是最不像机器学习的一种，它更像是一种控制算法「*不要在意这些说法，后面了解多了就知道了*」；众所周知，机器学习可以分为监督学习和非监督学习，我们常用的也大多是监督学习；而说到它和经典机器学习算法——监督学习最大的不同之处，也就是上面说到agent和environment交互时候，environment给出的「agent获取的」并不是确定的教师信号 而只是评价信号；

这也正是RL和SL(supervise learning)的区别所在，SL确定性的给出在不同状态下的教师信号 而SL的这个学习优化过程 其实更像是学习拟合到这个：从状态到教师信号的映射 的过程，而事实上对于一些复杂的控制问题来说，往往是需要分阶段考虑的，很难给出一个确定的 最优的 分解好的过程，用于其来学习这个映射关系 比如机械臂，「*联系后面的多步预测和单步预测的不同来理解*」；

这样的话 对于SL来说，直接学习到一个从初始状态到最终拿起东西的这样一个映射 成为了一个很困难的事情，毕竟SL 本质上也还是分类，当分类的类别都不清楚的时候 SL也就无能为力了；恩 这里我们就是说从状态分布到action之间的关系，action就是一个个的类别，为了实现目的来吧不同状态归于不同action，来实现决策的目的；于是就像是机械臂问题 就无法直接的找到一个action或者说类别来吧当前的state放入，完成任务；这里所说的action也还是low-level的action选择，真选择一个足够high-level的来完成任务，学习映射也可以，想想看这些low-level的action能够组成多少high-level吧，都差不多是能看做是continued的了，而且还可能没有常见continued action的内在关系；而对于RL不同，无论RL的算法如何，最终只有一个目的：让折扣累计的reward最大化；每个reward看做对应一个state或者一个action的选择 那么就完成一系列的action的选择了，如果把整体完成任务看做一个high-level action，这些一个个的选择看做了low-level的话，这也 可以看做就是RL和SL的区别了吧；

当然 其他的不同之处的话，还可以从其中的训练数据来进行理解，RL需要有reward的交互数据，而SL只是单纯的标签数据就好；

根据观测数据的时间特性 预测可以分为单步预测和多步预测「基于历史数据预测现在和未来的区别」；关于监督学习和时间差分的区别，**传统**监督学习是根据预测值和实际观测值的误差来修正预测模型，而TD是根据时间上连续两次预测之间的差值来修正预测误差；具体的来说 前者需要得到全部观测数据后 才能通过计算预测误差来修正预测模型，后者 只需要某两个时刻的预测值和局部观测数据来修正预测模型；所以可以实现在线学习 减少存储量和计算量 有着更高效的学习效率；同时 也可以看出 监督学习只能实现单步学习预测 也就是只能基于当前信息来对于当前时刻的输出进行预测，而TD可以实习多步预测； 

## 强化学习的分类

 基于学习系统和环境交互的类型，强化学习算法可以分为非联想强化学习(non-associative RL)和联想强化学习(associative RL)；前者的学习系统仅从环境中获取回报，而不区分环境的状态；

>   研究成果包括 Thathachar等针对n臂赌博机问题提出来的基于行为值函数的估计器方法和追赶方法(pursuit method)和SUtton提出的增强信号比较方法(reinforcement comparison)

后者在获取回报的同时，具有环境的状态信息反馈 类似于反馈控制系统; 同时可以进一步细分，也就是可以分为：即时回报联想强化学习和序贯决策强化学习两种；顾名思义 即时回报的就是强化学习的回报信号没有延迟特性，学习系统直接最大化期望的即时回报 比如联想搜索(associative search)、可选自主方法等；

进而后者的序贯决策强化学习方法，才是我们日常所常说的强化学习的方法，毕竟大部分实际问题都具有延迟回报的特点，进而在解决这样的序贯决策问题中 采用了markov决策过程模型；

>   这里也终于引出了markov建立的意义：是为了解决现实中序贯决策的实际问题，而建立的模型；

分类的话，根据学习目标 可以将其分为折扣性回报指标和平均回报指标两种；「最终目标式子的区别」

然后 根据markov决策过程行为选择策略的平稳性「概率分布的不变性」强化学习算法可以分为**求解平稳策略markov决策过程值函数**的学习预测「learning prediction」，和**求解markov决策过程中最优值函数和最优策略**的学习控制「learning control」

## prediction和control

这里想先区分一波关于prediction和control；

上面对于强化学习整体的分类有所介绍，紧接着对于我们经常需要解决的序贯决策问题 常常需要建立起来markov过程进行求解，这样的话 对于常见的强化学习问题可以分为学习预测和学习控制两类问题，前者的求解为实现以优化决策为目的的后者提供了基础；

具体的来说话，对于强化学习来说 一个绕不开的问题就是关于：计算平稳行为策略markov决策过程的状态值函数的问题「策略评估」；换句话说就是 在一个确定性的行为策略下如何确定其值函数；毕竟啊 确定好一个policy之后，也就相当于确定了一个markov链，那么如何计算这个markov链上的状态值函数也就成为了一大问题；

这时候就需要分：model-based 和model-free两种情况下了，「关于这两者的定义区别，直白的的说就是转移概率知不知道，也就是选取某个action后 能否直接推导下一个state 而不需要借助environment」对于model-based的情况下，直接借助动态规划就好了 具体可以分为：策略迭代、值迭代、策略搜索等；「已知模型了 那么就知道了转移概率 就可以直白的计算值函数；具体参考深入浅出P36」而实际问题中 更多的依旧是model-free 「某一个状态的之后状态的转移概率未知 那么就无法根据贝尔曼方程来直白的计算，于是我们就需要先知道当前这条markov链」于是对于这一类markov链的状态值函数的求解 也就是一类多步学习预测问题，常见的方法包括蒙特卡洛方法 时域差值学习；

首先是对于蒙特卡洛的首先进行介绍，简单的说就是基于经验（experience）进行学习，这个经验包括样本序列的状态（state）、动作（action）和奖励（reward）。得到若干样本的经验后，通过**平均所有样本的回报（return）**来解决强化学习的任务。

但显然能看出蒙特卡洛的方法，尽管可以作为一种类似监督学习的方法「基于历史数据 进行预测」可以解决多步预测的问题，但是有着效率低下的特点；而时域差分的方法 利用连续两个时刻预测值的差值来更新模型；

如果继续往下看的话，包括Qlearning、SARSA等控制学习算法的基础 也都是在这里，进而我们常见的强化学习中所分类得到的表格式和值函数逼近式的，也正是因为这里的TD算法的分类而导致的；进一步的还有对于单步预测和多步预测还有着：TD(0)、TD(λ)；具体的可以翻阅Reinforcement Learning: An Introduction ；

>   顺带说一句：根据观测数据的时间特性 预测可以分为单步预测和多步预测「基于历史数据预测现在和未来的区别」；关于监督学习和时间差分的区别，**传统**监督学习是根据预测值和实际观测值的误差来修正预测模型，而TD是根据时间上连续两次预测之间的差值来修正预测误差；具体的来说 前者需要得到全部观测数据后 才能通过计算预测误差来修正预测模型，后者 只需要某两个时刻的预测值和局部观测数据来修正预测模型；所以可以实现在线学习 减少存储量和计算量 有着更高效的学习效率；同时 也可以看出 监督学习只能实现单步学习预测 也就是只能基于当前信息来对于当前时刻的输出进行预测，而TD可以实习多步预测；
>
>   时域差分算法 往往被看组学习控制算法 如sarsa和qlearning的一部分 就像多步预测被看做为多步控制的一部分一样；

大致介绍了关于学习预测的内容之后，紧接着可以介绍关于学习控制的问题；

在上面也提到过学习预测可以看做学习控制的一个子问题，原因也就是在于 常见的强化学习算法的目的毕竟还是为了优化决策求解出来一个合适的行动策略；而学习预测本质上依旧是为了给学习控制提供出来一个评判标准：评判出来当前的优化得到的策略是好亦或是坏；这句话也指明了学习控制本质的作用：求解策略，而学习预测本质目的：求解出来一个评价函数；

>   通过上面的总结 我们可以直白的说 大部分强化学习都还是为了解决序贯决策问题而诞生的，而解决的方式就是先建立markov模型，同样的动态规划方法也可以用于解决markov模型 于是可以一起被用来解决序贯决策问题，不过后者需要提前知道模型情况 也就是状态转移概率，于是用来解决model-based问题；其余的RL算法被用来求解model-free的；可以这么说动态规划的思想是无模型强化学习的起源；这些也正是之间的关系：序贯决策→→markov决策→→动态决策/RL；「其实从markov决策出发 进而引到RL会更为直观」

所以我们知道 强化学习和动态规划是求解markov决策问题的两大方法，而这里的求解 也就是上面说的学习控制；类似于动态规划中的策略迭代需要先有着对于策略评价方法的策略评价，强化学习中 需要先得到一种适合的用于评价策略的方法 这也正是学习预测了，进而才有着类似动态规划中策略迭代一样的学习控制方法；