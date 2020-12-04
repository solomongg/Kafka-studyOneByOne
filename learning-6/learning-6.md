# Kafka 学习笔记

## Kafka  ISR/HW/LEO/LSO  概念理解

### 目录

- ISR
  
- HW
  
- LEO
  
- LSO
  
#### ISR

- 理解

  Kafka中特别重要的概念，指代的是AR中那些与Leader保持同步的副本集合。 在AR中的副本可能不在ISR中，但Leader副本天然就包含在ISR中。

- 原理

  ISR (in sync replica) , 与leader副本保持同步状态的副本集合。
  - Leader对IRS中的节点进行track，当follower挂掉、卡住、落后时会被移出ISR 。
  - 依据replica.lag.time.max.ms参数判断 stuck and lagging replicas。
  - 当所有ISR中副本都将某条消息写入log后，可以确定该消息已被提交，并且只有被已提交的消息才会被消费者消费到。
  
#### HW

- 概念
  HW（High watermark）：高水位值，这是控制消费者可读取消息范围的重要字段。一个普通消费者只能“看到”Leader副本上介于Log Start Offset和HW（不含）之间的所有消息。水位以上的消息是对消费者不可见的。

- HW更新
  
  - follower HW
    follower HW 更新遵从最开始说的那个规律，在日志成功写入，LEO更新之后，就会尝试更新自身HW的值的，这个尝试发生在收到FETCH响应时会比较本地HW值和leader中的HW值，选择小的作为自身的HW值，所以说follower的HW值。但是follower 的HW值，说实话并没有什么卵用，说到用处的话应该是为称为leader做准备吧。相对来说leader 的HW值才是业务中所关心的，它决定了consumer端可消费的进度。
  - producer 产生消息并且LEO成功更新
    HW的值可能会尝试更新（这需要根据ISR的同步策略来确定），然后还有leader在处理FETCH的请求时也会尝试更新。另外还有就是follower时、某个副本被提出ISR时都会尝试更新对应的HW值。这四种情形里面，最常见的就是接受FETCH请求时，通过比较自己的LEO值与缓存的其他的follower的LEO值，选择其中最小的LEO值来作为HW值，所以说HW值实际上就是ISR中最小的副本的LEO值啦
  - leader HW
    其中leader 是通过follower 的offset来确定follower上次的消息是否写入的，所以就导致，remote LEO是比leader LEO小1的（更新remote LEO更新的其实是上次操作的结果），这就导致了如果最后一个follower同步完成时，HW实际上是未被更新的，得等到第二次FETCH请求才能完成HW的的更新（也就是说 第一轮FETCH完成了消息的同步（但leader 对于folllower是否成功保存毫不知情），第二轮完成了消息同步的同时完成上一轮的HW值的更新）

#### LEO

- 概念
  日志末端位移值或末端偏移量，表示日志下一条待插入消息的位移值。举个例子，如果日志有10条消息，位移值从0开始，那么，第10条消息的位移值就是9。此时，LEO = 10。

#### LSO

- 概念
  这是Kafka事务的概念。如果你没有使用到事务，那么这个值不存在（其实也不是不存在，只是设置成一个无意义的值）。该值控制了事务型消费者能够看到的消息范围。它经常与Log Start Offset，即日志起始位移值相混淆，因为有些人将后者缩写成LSO，这是不对的。在Kafka中，LSO就是指代Log Stable Offset。

