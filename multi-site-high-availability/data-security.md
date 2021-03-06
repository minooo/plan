# 数据安全保障
数据安全是最为重要的点，如果无法保障数据的安全，那么后续的其他功能就失去了意义。  
首先数据复制的流程可以分为三种场景：
1. 每个 zone 之间正常复制。
2. 切换 zone 的时候数据复制。
3. 数据从零开始全量复制。

三种场景加上各种异常，还需要有一些容错的策略。PiPlan 在设计上是尽量避免这些问题的发生，然后是在无法避免的问题，在针对性的进行解决。这里主要介绍 PiPlan 设计思想和实现方式，没有以及的常见问题可以在[常见问题](/faq)里查看

## 保证任何时候一份数据只会被一个 Zone 写入
如果数据可以同时被两个 Zone 写入，那么面对数据冲突就无法处理，算法甚至人工都不一定能判断出来那条才是正确的。而且在任意一个时间点只允许被一个 Zone 写入并不会引起业务的问题。也能满足场景需要，因为在网关层会反向代理到正确的 Zone。所以设计了一系列策略来保障任何一个时间只会被允许一个 Zone 写入。
### 规则检查
在配置被提交之前就会对配置的规则进行检查，确保配置的每一个 ZSID 不会被一个以上的 Zone 判断为正确。验证方法为，每个 ZSID 尝试在每个 Zone 的进行检查，如果有一个以上的为通过，则配置检查失败。不允许提交。

### 第一次上线
第一次上线的时候，不管是一个 Zone 还是多个 Zone ，因为有规则检查，被允许提交的配置可以保障不会有一个以上的 Zone 允许写入。也没有切换等中间态，可以保障 Zone Sharding 的安全。

### Zone 切换
最有可能出现多个 Zone 写入的场景是 Zone 切换的时候，比如把 A 地区的用户从 Zone A 切换到 Zone B。这个过程需要保障新老两份规则不能同时生效，否则两个规则冲突，就会出现一个 ZSID 可以同时被两个 Zone 处理的场景。我们以 PiDAL 为例子，在切换 Zone Sharding 规则的时候是如何处理的，网关层等其他地方时实现的思路是一致的。假设场景为要把 ZSID 为 1 的请求从 Zone A 切换到 Zone B。  

#### 1. PiDAL 实例 Keepalive  
定时比如每 5 秒，就向 PiMMS 同步状态和对比自己的配置和 PiMMS 的是否一致，如不一样就切换为 PiMMS 的配置。在日常的状态下 Keepalive 机制也能尽早的发现 PiDAL 实例的问题并告警，比如连续 4 个周期没有收到 PiDAL 的请求，就认定 PiDAL 实例异常告警处理。如果 PiMMS 发现实例有问题，需要先处理掉问题之后才允许做变更操作。Keeplive 的检查周期和报警周期影响到服务不可用的最长时间，比如每 5 秒检查一次，4 个周期异常开始报警，就意味着最长 20 秒会才会发现问题。这个可以根据业务情况和 SLA 要求进行配置。

#### 2. 记录当前状态便于操作回滚  
在执行变更之前，记录下当前的状态，在遇到异常情况的时候可以及时回滚，保障数据安全和业务可用性。

#### 3. 推送变更记录 1: 把 Zone A 和 Zone B 里的 ZSID = 1 的分发状态置为 BLOCK。  
这条变更记录被应用之后效果为：禁止所有写操作，对所有写操作返回错误：Zone resharding block error。   
如果在指定时间内 PiDAL 实例都接收到请求并正确返回。就意味着所有实例已经接收到来变更，就可以继续下一步操作。但是这一步也可能出现意外，操作回滚。回滚时候是能保证数据安全的，针对每种异常情况的论证：

1. 变更记录没能到达某个实例、没能接收到这条变更  
这种情况下，执行回滚，把 ZSID = 1 的规则状态回滚为为 ACTIVE， PiMMS 会向所有的实例推送这个变更。如果这个实例能收到这个变更，数据和现在的状态一致，所以状态是正确的。如果在回滚的时候某个已经应用变更记录 1 的实例没能收到回滚请求的时候，这种情况下这个实例的状态为 BLOCK，是不会产生写请求的，所有数据是安全的，因为 PiDAL 的 Keepalive 机制，这个实例的状态也会最终保持和 PiMMS 一致。

2. 实例已经挂掉  
执行回滚和 3.1 的场景一样，当这个挂掉的实例在恢复的时候会从 PiMMS 从新拉取一份元数据，这就会保持数据一致了。

3. 实例接收到变更记录但是 PiMMS 没能正常收到响应。  
执行回滚，和 3.1 的异常处理一样，就算在回滚的时候，有个别实例没能正常接收到请求，Keepalive 机制也会保障最终状态一致。

4. 实例正常处理了变更记录 1 ，而且 PiMMS 也正常接收到了变更结果，但是之后实例挂了  
这种情况在第一步是无法发现的，因为第一步已经结束了。但是这种 case 会在 第二步被发现并被处理。

#### 4. 推送变更记录 2: 把 Zone A 和 Zone B 里的 ZSID = 1 规则置为由 Zone B 处理。并将状态置为 RESHARDING。  
这一步开始切换配置，这条变更被应用后，所有实例规则变更，而且依然禁止写操作。对写操作返回错误：Zone resharding block error。如果所有实例正确影响配置后，可以继续执行下一步，如果有异常情况，执行回滚。这个时候能保证数据是安全的。

> 这一步执行完成后记录下切换的时间，后续逻辑机制会用到。如果对本地时间漂移问题很敏感可以使用 [PiLCS](/pilcs/introduction) 解决此类问题。

1. 变更记录没能到达某个实例或者实例挂掉  
网络丢包或者正常处理完第一步的时候实例挂掉了，到这一步的时候就会被发现，这个时候执行回滚。因为上一步的时候已经确保所有的实例中已经把 ZSID = 1 的状态变更为 BLOCK。能正常接收到回滚请求的实例，都会回滚，那些回滚失败或者挂掉的实例是也不会产生写请求，数据也是安全的。

2. 实例接收到变更记录但是 PiMMS 没能正常收到响应  
和 3.3 的 case 一样，正常回滚的实例都可以正常处理，那些回滚失败的也不会产生写入，所以依然是安全的。Keepalive 机制会保障这些回滚失败的实例会恢复到一致的状态。

3. 正常处理了变更记录 2 但是之后实例挂了  
和变更记录 1 一样，也是等在下一个步骤的时候被发现和处理掉。

#### 5. 推送变更记录 3: 把 Zone A 和 Zone B 里的 ZSID = 1 规则置为状态置为 ACTIVE。  
这一步需要确保 Zone A 里面的数据已经全量同步到来 Zone B，因为之前已经阻塞来新的写入，所以这一步通常会在秒级别完成。应用这条变更记录后让新配置生效，变更被应用后，ZSID = 1 规则只允许在 Zone B 写入。并且开放 ZSID = 1 的写入。这一步遇到异常不再是回滚，而是重试，在出现异常和重试的过程中依然是安全的。

1 实例没有接收到变更记录 3  
没有接收到变更记录的推送的实例，状态依然是 RESHARDING，这一状态，只是会停止写入，不会造成其他 Zone 的数据写入。通过 Keepalive 机制实例状态会调整成 ACTIVE。

2 实例已经挂掉  
挂掉的实例已经无法处理业务了，所以也不会对造成不安全的问题。，而且通过 Keepalive 机制，会及时发现这个有问题的实例。

3 实例接收到变更记录但是 PiMMS 没有接收到响应  
此时实例的数据状态已经是正常了，即便是重试或者 Keepalive 都会被幂等掉。只是 PiMMS 不知道这个实例已经正常了而已，Keepalive 会让 PiMMS 获得实例的最新状态。

4 正确处理后，实例挂掉  
这个场景和正常运行的时候挂掉一样，并不会因为刚做了 Zone Sharding 的切换而引起什么问题。通过 Keepalive 机制可以发现挂掉的实例，甚至自动拉起恢复。

#### 为什么会有 RESHARDING 状态
上述的步骤中在推送变更记录 3 的时机是需要等待数据已经被全量同步，如果数据没有同步完成，就会造成 Zone B 的读取数据落后，如果有更新老的数据甚至会造成数据冲突。切换到 RESHARDING 状态之后记录时间 T，T 时间之后可以确保所有的实例都不会在产生写入请求，假设每个 SQL 的执行超时时间为 1 秒，那么 T + 1 之后不会在有 ZSID = 1 的数据写操作了。可以通过这个时间来判断数据同步状态。


## 数据同步的安全保证
在 Zone Sharding 切换的时候提到需要保障数据同步完成之后才可以完成最后异步切换。数据同步安全需要保障两方面安全：
1. 数据防回环。
2. 变更不能丢失。
3. 变更顺序不会错乱。
在保障这些基本要求之上需要对尽量低的延迟比如 1 秒延迟。虽然负责跨 Zone 同步消息的 MQ 可以保证数据和顺序。但是生产环境和网络环境错综复杂，还有各种场景和 BUG 的存在，所以数据同步还需要具备容错能力，在实现的时候不能完全依赖 MQ 来保障消息安全和顺序。PiDTS 的数据同步通过 LSN 校验、同步幂等两个核心概念来保障数据同步的安全。

### 数据防止回环
防止数据回环是数据同步中最重要的一点，即便能保证变更消息不会丢失、顺序也不会错乱数据回环也会引起问题。数据回环不仅仅是在 Zone Sharding 切换的时候需要做，正常的数据复制也需要防止数据回环。  
数据回环是指在一个双向/多向同步的数据库集群中，在数据库 A 中变更了数据，复制到另外一个数据库 B 中，依然会产生一个变更日志，此时数据库 B 的数据再次同步回数据库 A，如此反复循环下去。在数据新增、变更、删除三种情况下数据回环都会引起问题。以 MySQL 为例解释下数据回环引起的问题。
#### 新增数据
在数据库 A 中新增一条数据 `name = "q",age = 20`，此时会产生一条 binlog 并同步 给数据库 B，数据库 B 应用这条 binlog 除了同步了这条记录外同时也产生了一条新增的 binlog ，在双向同步的时候这条 binlog 记录会被同步给数据库 A，如果这个表有唯一性约束就会造成写入失败，如果没有唯一性约束就会造成重复。
#### 更新数据
此时数据库 A 和数据库 B 都有一条记录 `age = 20`，此时数据库 A 变更 `age = 21` 同时也会产生一条 binlog 并同步给数据库 B，数据库应用这条日志之后也会产生一条一样的 binlog 并同步给数据库 A，因为数据库 A 里面 已经是 `age = 21` 了，所以这次应用 binlog 并不会因为改变，也不会产生 binlog，数据也不会回环。

但是如果数据库 A 同时变更了两次情况就会完全不同了，数据库 A 先是 `age = 21`，然后又变更了一次数据 `age = 22`。这样就会产生两条 binlog 同步给数据库 B。数据库 B 应用这两条 binlog 之后，也会产生两条 binlog，顺序也是先 `age = 21` 然后 `age = 22`。当第一条 binlog 同步给数据库 A 的时候，A 会被设置为 `age = 21`，然后第二条 binlog 会被变更为 `age = 21`，这样有产生了两条 binlog 并同步给数据库 B，如此反复循环下去。
#### 删除数据
删除和更新比较类似，在先新增后删除连续两个操作的情况下，也会和更新一样陷入「新增 -> 删除 -> 新增」 的无限循环下去。
#### 解决方案
PiDTS 通过增加在数据库表中增加一个字段来解决回环问题，搭配 PiDAL 在数据变更的时候自动维护这个字段，能做到业务无感知。
在这个新增的字段里，包含产生此次一次变更发生的 Zone ID ，在数据同步的时候会一起同步出去，当这个数据被同步回来的时候，通过对比 Zone ID 如果和本 Zone ID  一致就丢弃，进而防止数据回环的发生。整体原则可以概括为，「**谁负责写入同步谁产生的数据**」。
<!--
Easter egg：
这个方案有个 Bug，就是在删除的时候，比如在 Zone A 中先删除 id = 1 的数据，然后数据还在同步的时候 Zone A 新增了一条 id = 1 的新数据，那么有可能这个新数据会被删除。
不过这个问题也是有解决方式的，就是 Replicator 在处理 Delete 操作的时候，通过被删除前的数据和 Zone Sharding 规则实时计算出应该是那个 Zone 删除的，如果不是当前 Zone 负责处理这条数据，就不广播 Delete 的 ChangeEvent，接口避免。
-->
#### 三个及以上 Zone 的情况
如果是两个 Zone 互相同步就比较好判断，当三个及以上 Zone 的情况就需要额外处理了。假设有 A、B、C 三个 Zone ，就以 ZSID = 1 的数据同步情况为例。

![三个 Zone 同步](../static/pidts/data-sync.png)

由[保证任何时候一份数据只会被一个Zone写入](./保证任何时候一份数据只会被一个Zone写入) 我们可以保障任何一个时间只会被一个 Zone 写入。假设此时 ZSID = 1 被 Zone A 写入，Zone B 收到同步请求之后会应用到本地，然后各自 binlog 被同步给 Zone C 和 Zone A，Zone A 判断是自己本地产生的变更，会忽略这条 binlog。Zone C 判断这条数据应该由 Zone A 写入，但是 binlog 是 Zone B 同步过来的，也会忽略这条 binlog，由此在三个及以上的 Zone 的情况也能保障同步的安全。


### PiDTS 组件和核心概念
#### ChangeEvent 和 LSN
ChangeEvent 和 LSN 是 PiDTS 的主要对象，其中原始数据库的每次变更会被转化为 PiDTS 里的 ChangeEvent，这个概念屏蔽里不同数据库之间的差异，在处理的时候可以忽略掉原始数据库的差异。而 LSN 是每个事件的序号，用来标示每条日志。

ChangeEvent 和 LSN的定义：
```python
class ChangeEventLSN(object):
    def __init__(self):
        self.source_zone_change_no: int = 0
        self.server_id: int = 0  # 产生变更的机器 ID
        self.log_index: int = 0  # 日志文件的 编号
        self.log_position: int = 0  # 日志位置
        self.xid: int = 0
        ...... 


class ChangeEvent(object):

    def __init__(self):
        self.lsn: ChangeEventLSN  # 日志序列号
        self.prev_lsn: ChangeEventLSN  # 上一条数据的 LSN
        self.source_zone_id: int  # 产生变更事件的 Zone ID
        self.old_data: dict  # 变更前数据
        self.new_data: dict  # 变更后数据
        self.is_retry: bool  # 是否是在回滚/重试
        ......
```
`ChangeEventLSN` 是由根据固定信息生成的。同一条原始信息可以生成一样的 LSN，而且通过 LSN 也可以反推到原始格式的位置便于使用。LSN 是单调递增的，通过 LSN 对比也能知道消息的先后顺序。  
`ChangeEvent` 不仅包含自己的 LSN，还包含了上一条的 LSN，这样可以在收到消息的时候来验证是否论序或者丢失消息。

#### PiDTS Replicator 和 Apply
Replicator 是负责将原始变更事件转换为 `ChangeEvent` 并推送给队列的组件，例如在对应 MySQL 的 Replicator 里，负责把 binlog 日志转化为 `ChangeEvent` 并推送给队列。`Apply` 负责从队列里接收 `ChangeEvent` 并进行校验，通过之后写入到目标数据库。

### 数据同步顺序和是否丢失保障
Replicator 是从已经持久化数据库实例读取日志，所以读取的时候顺序有保障的，也不会丢失消息，除非 binlog 已经被删除了。这个时候通过[全量复制]()依然可以得到完整和准确的数据。

在 Apply 接收到消息的时候会校验 `prev_lsn` 是否和收到的上一个 `lsn` 判断是否一致，如果不一致有三种情况：
1. 消息乱序
2. 消息丢失
3. 消息重发

针对乱序和丢失，会停止应用到目标数据库上，然后 DTS 会自动尝试重发一段时间之前的日志，如果重试后还不行，就停止复制，因为无法保证数据的正确性，给出报警，及时排查和处理。

而面对消息重发的情况又会有两种情况：
1. 当前消息的`ChangeEvent` 因为某些原因会重复发来两次，因为 lsn 是唯一的，只需要判断 lsn 和上一个是否相等就可以做幂等处理。
2. 从已经接收到的消息之前的一段距离的日志重新发送。针对这种情况 Replicator 在发送的时候会加上 `is_retry: True` 这样 Apply 会忽略此次的 prev_lsn 校验。重复的消息会有幂等机制保障不会被老的数据覆盖掉。

### 数据同步幂等
消息的重发不论是重试还是等其他场景下，都是会作为一种保障的方式存在的。针对消息重发这种场景不能发生老的数据覆盖已经同步过来的新数据，这样就会造成数据异常。比如：

| 消息序号 | 值 |
| :-- | :-- |
| 1 | 100 |
| 2 | 101 |
| 3 | 102 |
| 4 | 103 |

最终的结果是 `103` 然后这个时候收到重发的消息

| 消息序号 | 值 |
| :-- | :-- |
| 2 | 101 |
| 3 | 102 |
| 4 | 103 |

在处理序号为 2、3、4 的时候需要一直保证值为 `103` ，如果发生变换比如 `101` -> `102` -> `103` 这样最终还是 `103` 但是，在 `101` -> `102` 的期间，值并不是 `103` ，这个时候如果有读取的行为就会读到错误的数据。所以幂等也需要保障数据的安全。

PiDTS 通过增加一个字段来确保幂等操作，如果单独 PiDTS 的时候可以给数据库表增加一个自动更新的字段
```
`update_time` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6)
```
这样 update_time 字段在每次变更的时候都会随着时间增长。如果业务场景要求很敏感不能依赖时间戳，
可以搭配 PiDAL 会自动插入一个连续递增的序号，在遇到时间漂移或者无法区分时间精度的场景下，依然能保持递增有序。
Apply 在应用 ChangeEvent 的时候会判断目标数据库是否大于 ChangeEvent 里的 update_time，只有大于的情况下才会被应用到目标数据库。这样在面对消息重发的时候幂等机制能保障数据的安全。

### 容错能力
虽然在正常的情况下可以保障不会出现问题，但是在错综复杂的网络环境中依然会产生各种意外，所以也要具备一定的容错能力。比如在遇到业务 Bug、网络突然不可用等场景下，需要能最大程度的保障数据安全和系统的正常运行。 PiDTS 在面对复杂的错误情况下提供给了数据锁定和在线全量同步两种机制来保障遇到问题的是尽量减少异常带来的影响。
#### 数据锁定
在数据同步的时候，PiDTS 会在保证来顺序，幂等来保证重复等，即便全部通过前面的策略之后，在最后还会根据 old_data 进行校验，正常情况下，old_data 和数据库当前记录应该是一致的，如果无法通过校验就会锁定当前数据并报警，PiDTS 会进行校验，被锁定的数据无法被更改，一直到数据被人工修复，根据时间的经验来说除了手动触发的，自然触发数据锁定的概率大约是百万分之一。
#### 消息延迟
生产环境中需要尽量保证延迟尽量低，那么造成消息延迟可能有两种原因：
1. 负载压力大
2. 网络拥堵

##### 负载压力大
对于负载压力大的情况，只需要通过减小一个节点的压力即可，比如一个 Replicator 负责同步一个表，这种情况下，单表既然都能负责业务的查询和写入，因为 Replicator 和Apply 的逻辑相比业务来说简单那些校验逻辑也只是简单的数字对比而已。常规情况下一套 Replicator -> Apply 可以负载多个表甚至一个 Zone 的同步。

##### 网络拥堵
如果是因为网络拥堵造成的那就只能通过优化网络环境的方式来进行来， PiDTS 并不会给网络带来很大的压力。优化网络配置之后就可以改善网络环境造成的消息延迟问题。

#### 消息过期被丢弃
在一些场景下，一个 Zone 停止服务来一段时间，因为积累的消息太多，最早的消息被删除来，无法通过消息队列来进行同步，这种场景下 PiDTS 支持幂等的在线全量同步。不仅在这种场景下，在新增一个 Zone 的时候等场景下、在一个 Zone 数据出现严重问题的时候需要都可以不影响其他 Zone 正常运行的情况下进行数据同步。PiDTS 的在线全量同步也是可以保障数据安全的。

### 在线全量同步安全保障
PiDTS 的在线全量同步会同时进行两个操作：
1. Row Copy
2. Incremental Copy

#### Row Copy
Row Copy 是复制已经存在的老数据，通过 `select * from xxx order by id` 不断的获取 src Zone 的数据，直到原始数据全部被复制完 Row Copy 操作结束。src Zone 的数据在同步到 dest Zone 的时候会被转换为 `insert ignore into` 操作。在 Row Copy 期间产生的数据变更会通过 Incremental Copy 操作被同步到 dest Zone 不会丢失。

#### Incremental Copy
Incremental Copy 是实时复制通过日志变更记录来进行同步，也就是 PiDTS 正常情况下运行的数据复制模式。Incremental Copy 和 Row Copy 是可以同时进行的。 Incremental Copy 会把数据变更记录也做一次转换，会把原始的 `insert` 转换为 `insert or update`、 `update` 会根据情况转变为 `insert or update`、`delete` 操作保持不变。**在全量复制的时候以 Incremental Copy 复制的数据为准**。

#### 安全保障论证
Row Copy 和 Incremental Copy 同时进行是可以保障数据不会丢失，最终的数据状态也会和 src Zone 一致。针对数据的 `insert`、`update`、`delete` 的三种操作逐一进行说明。

##### Insert
Row Copy 和 Incremental Copy 无论顺序是先后，都是会被插入的，所以 Insert 数据都是安全的。

##### Update
在全量同步的时候进行了 Update 操作，说明这条数据已经存在了。无论哪种方式总归是会被同步过去的。只是顺序的问题而已，如果 Update 操作发生在 Row Copy 之前，那么这个最新的数据会通过 Row Copy 的方式同步给 dest Zone。

如果是 Row Copy 先到达直接进行插入即可，此时数据就是最新的，如果这之后 Incremental Copy 在同步变更事件过来，就执行正常的 update 即可，因为数据是一样，不会有数据变更。 如果是 Incremental Copy 先到达会转换为 insert 操作，这个时候插入的就是最新的数据，而后续 Row Copy 到达会因为幂等机制而忽略。

如果被 Update 的数据行是先被 Row Copy 操作来之后发生的，因为 Row Copy 和 Incremental Copy 是同时进行的，那么对应的修改记录会通过 Incremental Copy 操作被同步过来。此时因为数据已经存在了，进行正常的 update 操作即可。

##### Delete
Delete 操作相对 Update 来说简单一些，如果在 Row Copy 之前进行了 Delete 操作，那么 Row Copy 就不会同步这一条数据。如果是在 Row Copy 之后进行的 Delete 操作，那么最终会被 Incremental Copy 同步过来 Delete 操作。

### Zone 断电、断网
介绍了那么多保障和容错措施，那么 PiPlan 在面对突然断电的或者断网的场景会如何处理。

首先需要了解面对 Zone 断电断网的时候会发生什么。虽然 PiDTS 同步效率很高，但是及时只有 1 秒的延迟，在面对断网断电的时候，也会有 1 秒的数据没有被同步。

![断网断电](../static/pidts/data-security-three-zone-dts.png)
