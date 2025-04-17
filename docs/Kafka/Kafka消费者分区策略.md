# Kafka消费者分区策略

### 消费者分区策略

可以通过消费者参数

**partition.assignment.strategy**

设置分区分配给消费者的策略。默认为Range。允许自定义策略。

#### **Range（范围）**

把主题的连续分区分配给消费者。（如果分区数量无法被消费者整除、第一个消费者会分到更多分区）

**对每个Topic进行独立的分区分配**，首先对分区按照分区ID进行排序，然后订阅这个Topic的消费组的消费者再进行排序，尽量均衡地将分区分配给消费者。这里的“尽量均衡”是因为分区数可能无法被消费者数量整除，导致某些消费者可能会多分配到一些分区

它的特点是以topic为主进行划分的，通过partition数/consumer数来决定每个消费者消费几个分区。如果有余则交给消费者1

假设消费者数量为N，主题分区数量为M，则有当前主题分配数量 = M%N==0? M/N +1 : M/N ;

简单来说就是将主题中的分区除以group中订阅此主题的消费者，除数有余则一号多分配。

![1737698386440-6570cf2b-5cd3-4cd1-a61c-b43b25823387.png](./img/5ab67-dCWffn22Ll/1737698386440-6570cf2b-5cd3-4cd1-a61c-b43b25823387-457076.png)

Range策略的缺点在于如果Topic足够多、且分区数量不能被平均分配时，会出现消费过载的情景，举一个例子

![1737698386482-409ac9d7-b4aa-4781-badc-81d1978574bc.png](./img/5ab67-dCWffn22Ll/1737698386482-409ac9d7-b4aa-4781-badc-81d1978574bc-786451.png)

可以看到此种情况已经相差3个分区，如果主题进一步扩大差距会愈发明显。

#### **RoundRobin（轮询）**

把主题的分区循环分配给消费者。

一种轮询式的分配策略，即每个人都会得到一个分区，顺序取决于他们注册时的顺序。这有助于确保所有消费者都能访问到所有数据。

简单来说就是把所有partition和所有consumer列出来，然后按照hashcode排序，最后进行轮询算法分配。

如果主题中分区不一样的时候如下：

![1737698386482-bc0967df-0058-47e0-847f-1fec0e2198a9.png](./img/5ab67-dCWffn22Ll/1737698386482-bc0967df-0058-47e0-847f-1fec0e2198a9-715144.png)

不难看出轮询策略是将partition当做最小分配单位，将所有topic的partition都看作一个整体。然后为消费者轮询分配partition。当然得到此结果的前提是Consumer Group种的消费者订阅信息是一致的，如果订阅信息不一致，得到的结果也不均匀，下面举个例子：

![1737698386465-61354ddf-dec8-4290-bd70-189205b0009b.png](./img/5ab67-dCWffn22Ll/1737698386465-61354ddf-dec8-4290-bd70-189205b0009b-434887.png)

如图，Consumer0订阅Topic-A、B，Consumer1订阅Topic-B、C

顺序注意图中的Seq，先分配TopicA

第一轮 : Consumer-0: Topic-A-Partition0

由于Consumer-1没有订阅Topic-A，所以只能找到Topic-B给Consumer-1分配

于是 Consumer-1: Topic-B-Partition0

---

第二轮: Consumer-0: Topic-A-Partition0,**Topic-A-Partition1**

Consumer-1: Topic-B-Partition0,**Topic-B-Partition1**

---

第三轮: Consumer-0: Topic-A-Partition0,Topic-A-Partition1，**Topic-A-Partition2**

Consumer-1: Topic-B-Partition0,Topic-B-Partition1，**Topic-B-Partition2**

---

第四、五、六轮：

Consumer-0: Topic-A-Partition0,Topic-A-Partition1，Topic-A-Partition2

Consumer-1: Topic-B-Partition0,Topic-B-Partition1，Topic-B-Partition2,**Topic-C-Partition-0,Topic-C-Partition-1,Topic-C-Partition-2**

可以看到Consumer-1多消费了3个分区。所以在Consumer Group有订阅消息不一致的情况下，我们尽量不要选用RoundRobin。

注意：上面介绍的两种分区分配方式,多多少少都会有一些分配上的偏差, 而且每次**重新分配**的时候都是把所有的都**重新来计算并分配**一遍, 那么每次分配的结果都会偏差很多, 如果我们在计算的时候能够考虑上一次的分配情况,来尽量的减少分配的变动,这样我们将尽可能地撤销更少的分区，因为撤销过程是昂贵的

#### StickyAssignor(粘性)

粘性分区：每一次分配变更相对上一次分配做最少的变动.

当某个消费者的某个分区出现故障或不可用时，它会尝试保留它已经分配但尚未处理的分区。如果其他消费者也出现了问题，则会尽力维持原先的分区分配。这意味着一旦某个分区被分配给了某个消费者，除非该消费者退出或者分区不可用，否则不会重新分配给其他消费者。

目标：

1、**分区的分配尽量的均衡，分配给消费者者的主题分区数最多相差一个；**

2、**每一次重分配的结果尽量与上一次分配结果保持一致**

当这两个目标发生冲突时，优先保证第一个目标

首先, **StickyAssignor粘性分区**在进行分配的时候,是以**RoundRobin的分配逻辑来计算的,但是它又弥补了RoundRobinAssignor的一些可能造成不均衡的弊端。

比如在讲RoundRobin弊端的那种case, 但是在StickyAssignor中就是下图的分配情况

把RoundRobinAssignor的弊端给优化了

![1737698386440-25c26563-92d9-447c-91f3-198d74e5cb52.png](./img/5ab67-dCWffn22Ll/1737698386440-25c26563-92d9-447c-91f3-198d74e5cb52-574855.png)

此时的结果明显非常不均衡，如果使用Sticky策略的话结果应该是如此：

![1737698387019-5fa2413b-0458-45c8-8d00-38e1a61b67d4.png](./img/5ab67-dCWffn22Ll/1737698387019-5fa2413b-0458-45c8-8d00-38e1a61b67d4-967523.png)

在这里我给出实际测试结果参考

比如有3个消费者（C0、C1、C2）、4个topic(T0、T1、T2、T3)，每个topic有2个分区（P1、P2）

![1737698387210-8adc9209-ce1e-4692-bdf6-47f8e8a77df7.png](./img/5ab67-dCWffn22Ll/1737698387210-8adc9209-ce1e-4692-bdf6-47f8e8a77df7-212836.png)

**C0:****T0P0、T1P1、T3P0**

**C1: T0P1、T2P0、T3P1**

**C2: T1P0、T2P1**

如果C1下线 、如果按照RoundRobin

![1737698387198-4a952bd3-3433-4bfe-802f-2ec4956e69f5.png](./img/5ab67-dCWffn22Ll/1737698387198-4a952bd3-3433-4bfe-802f-2ec4956e69f5-467683.png)

**C0:****T0P0、T1P0、T2P0、T3P0**

**C2: T0P1、T1P1、T2P1、T3P1**

对比之前

![1737698387219-71f04c56-a167-4fa9-813e-8bacd7ba0124.png](./img/5ab67-dCWffn22Ll/1737698387219-71f04c56-a167-4fa9-813e-8bacd7ba0124-540012.png)

如果C1下线 、如果按照StickyAssignor

![1737698387219-c1949ab6-dbac-4e9b-9c65-d2109b393280.png](./img/5ab67-dCWffn22Ll/1737698387219-c1949ab6-dbac-4e9b-9c65-d2109b393280-381683.png)

**C0:****T0P0、T1P1、T2P0、T3P0**

**C2: T0P1、T1P0、T2P1、T3P1**

对比之前

![1737698387423-083016cc-eb27-45e1-90df-841072232608.png](./img/5ab67-dCWffn22Ll/1737698387423-083016cc-eb27-45e1-90df-841072232608-996654.png)

![1737698387618-031a6cb0-e8ae-4a80-a39f-374f67aa0e77.png](./img/5ab67-dCWffn22Ll/1737698387618-031a6cb0-e8ae-4a80-a39f-374f67aa0e77-789171.png)

#### 自定义策略

extends 类AbstractPartitionAssignor，然后在消费者端增加参数：

properties.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,类.class.getName());

即可。

### 消费者分区策略源码分析

接着上个章节的代码。

![1737698387649-76de593e-3659-490c-a7c3-2a65c066fe60.png](./img/5ab67-dCWffn22Ll/1737698387649-76de593e-3659-490c-a7c3-2a65c066fe60-454427.png)

![1737698387663-d935abaf-6b26-42ed-889f-3e7e97c55e79.png](./img/5ab67-dCWffn22Ll/1737698387663-d935abaf-6b26-42ed-889f-3e7e97c55e79-670567.png)

![1737698387687-be16f452-38a5-4a6a-ae98-828eea5220ff.png](./img/5ab67-dCWffn22Ll/1737698387687-be16f452-38a5-4a6a-ae98-828eea5220ff-479692.png)

![1737698387840-433e39b0-ffdf-4937-b274-e4d0e40123c1.png](./img/5ab67-dCWffn22Ll/1737698387840-433e39b0-ffdf-4937-b274-e4d0e40123c1-183886.png)

![1737698388054-fbc4879b-e659-4bf8-b344-4c9364d412a9.png](./img/5ab67-dCWffn22Ll/1737698388054-fbc4879b-e659-4bf8-b344-4c9364d412a9-567412.png)

## Consumer拉取数据

这里就是拉取数据，核心Fetch类

![1737698388084-8c62124d-35c1-4331-9ca3-25eaf66767e8.png](./img/5ab67-dCWffn22Ll/1737698388084-8c62124d-35c1-4331-9ca3-25eaf66767e8-993587.png)

![1737698388124-8a0b3e0b-2b72-411a-a3b5-0e7965b8d2c1.png](./img/5ab67-dCWffn22Ll/1737698388124-8a0b3e0b-2b72-411a-a3b5-0e7965b8d2c1-244289.png)

![1737698388121-022b53b2-074e-4d95-8b18-661506694506.png](./img/5ab67-dCWffn22Ll/1737698388121-022b53b2-074e-4d95-8b18-661506694506-117382.png)

## 自动提交偏移量

![1737698388195-8ff532a6-8ca5-4c80-a1fc-1f1c35b36cdf.png](./img/5ab67-dCWffn22Ll/1737698388195-8ff532a6-8ca5-4c80-a1fc-1f1c35b36cdf-749830.png)

![1737698388425-dcfcd91d-5908-49bd-8eaf-ad3f6bc230da.png](./img/5ab67-dCWffn22Ll/1737698388425-dcfcd91d-5908-49bd-8eaf-ad3f6bc230da-105616.png)

![1737698388564-9b3e87f0-60e8-4132-bc8b-8ac4ac7c26ac.png](./img/5ab67-dCWffn22Ll/1737698388564-9b3e87f0-60e8-4132-bc8b-8ac4ac7c26ac-497651.png)

![1737698388572-a4710820-852a-4af9-9462-c38da3f0630d.png](./img/5ab67-dCWffn22Ll/1737698388572-a4710820-852a-4af9-9462-c38da3f0630d-973475.png)

![1737698388574-3e7f8da7-7bf5-4de0-8cc2-e19fc07e3b7b.png](./img/5ab67-dCWffn22Ll/1737698388574-3e7f8da7-7bf5-4de0-8cc2-e19fc07e3b7b-608977.png)

![1737698388673-489f6ff6-30b2-4ee2-865f-f56cee35504e.png](./img/5ab67-dCWffn22Ll/1737698388673-489f6ff6-30b2-4ee2-865f-f56cee35504e-764329.png)

![1737698388841-1c40369e-f3ae-42d0-8ca6-d6d93de5f7ef.png](./img/5ab67-dCWffn22Ll/1737698388841-1c40369e-f3ae-42d0-8ca6-d6d93de5f7ef-312447.png)

![1737698389000-dad7af49-e063-44f3-924e-b2de7e44d5b5.png](./img/5ab67-dCWffn22Ll/1737698389000-dad7af49-e063-44f3-924e-b2de7e44d5b5-996415.png)

![1737698388993-86eb5c39-f464-428b-9572-97cc4dae31ed.png](./img/5ab67-dCWffn22Ll/1737698388993-86eb5c39-f464-428b-9572-97cc4dae31ed-189239.png)

当然，自动提交auto.commit.interval.ms

![1737698389103-7cbba0b2-dc2a-4008-9621-9936fe594644.png](./img/5ab67-dCWffn22Ll/1737698389103-7cbba0b2-dc2a-4008-9621-9936fe594644-684701.png)

默认5s

![1737698389075-5629b034-2c06-41e4-9717-b42374e13d4d.png](./img/5ab67-dCWffn22Ll/1737698389075-5629b034-2c06-41e4-9717-b42374e13d4d-957938.png)

从源码上也可以看出

maybeAutoCommitOffsetsAsync 最后这个就是poll的时候会自动提交，而且没到auto.commit.interval.ms间隔时间也不会提交，如果没到下次自动提交的时间也不会提交。

这个autoCommitIntervalMs就是auto.commit.interval.ms设置的
