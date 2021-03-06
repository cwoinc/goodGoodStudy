###zk实现分布式锁
使用zk实现分布式锁需要注意羊群效应，改进方案类似一致性哈希。
```
转自：http://jm.taobao.org/2012/10/25/zookeeper-herd-effect/

zookeeper中节点的创建类型有4类，这里我们重点关注下临时顺序节点。这种类型的节点有几下几个特性：

节点的生命周期和客户端会话绑定，即创建节点的客户端会话一旦失效，那么这个节点也会被清除。
每个父节点都会负责维护其子节点创建的先后顺序，并且如果创建的是顺序节点（SEQUENTIAL）的话，父节点会自动为这个节点分配一个整形数值，以后缀的形式自动追加到节点名中，作为这个节点最终的节点名。
利用上面这两个特性，我们来看下获取实现分布式锁的基本逻辑：

客户端调用create()方法创建名为“locknode/guid-lock-”的节点，需要注意的是，这里节点的创建类型需要设置为EPHEMERAL_SEQUENTIAL。
客户端调用getChildren(“locknode”)方法来获取所有已经创建的子节点，同时在这个节点上注册上子节点变更通知的Watcher。
客户端获取到所有子节点path之后，如果发现自己在步骤1中创建的节点是所有节点中序号最小的，那么就认为这个客户端获得了锁。
如果在步骤3中发现自己并非是所有子节点中最小的，说明自己还没有获取到锁，就开始等待，直到下次子节点变更通知的时候，再进行子节点的获取，判断是否获取锁。
释放锁的过程相对比较简单，就是删除自己创建的那个子节点即可。

问题所在
上面这个分布式锁的实现中，大体能够满足了一般的分布式集群竞争锁的需求。这里说的一般性场景是指集群规模不大，一般在10台机器以内。

不过，细想上面的实现逻辑，我们很容易会发现一个问题，步骤4，“即获取所有的子点，判断自己创建的节点是否已经是序号最小的节点”，这个过程，在整个分布式锁的竞争过程中，大量重复运行，并且绝大多数的运行结果都是判断出自己并非是序号最小的节点，从而继续等待下一次通知——这个显然看起来不怎么科学。客户端无端的接受到过多的和自己不相关的事件通知，这如果在集群规模大的时候，会对Server造成很大的性能影响，并且如果一旦同一时间有多个节点的客户端断开连接，这个时候，服务器就会像其余客户端发送大量的事件通知——这就是所谓的羊群效应。而这个问题的根源在于，没有找准客户端真正的关注点。

我们再来回顾一下上面的分布式锁竞争过程，它的核心逻辑在于：判断自己是否是所有节点中序号最小的。于是，很容易可以联想的到的是，每个节点的创建者只需要关注比自己序号小的那个节点。

改进后的分布式锁实现
下面是改进后的分布式锁实现，和之前的实现方式唯一不同之处在于，这里设计成每个锁竞争者，只需要关注”locknode”节点下序号比自己小的那个节点是否存在即可。实现如下：

客户端调用create()方法创建名为“locknode/guid-lock-”的节点，需要注意的是，这里节点的创建类型需要设置为EPHEMERAL_SEQUENTIAL。
客户端调用getChildren(“locknode”)方法来获取所有已经创建的子节点，注意，这里不注册任何Watcher。
客户端获取到所有子节点path之后，如果发现自己在步骤1中创建的节点序号最小，那么就认为这个客户端获得了锁。
如果在步骤3中发现自己并非所有子节点中最小的，说明自己还没有获取到锁。此时客户端需要找到比自己小的那个节点，然后对其调用exist()方法，同时注册事件监听。
之后当这个被关注的节点被移除了，客户端会收到相应的通知。这个时候客户端需要再次调用getChildren(“locknode”)方法来获取所有已经创建的子节点，确保自己确实是最小的节点了，然后进入步骤3。

为什么需要再次获得所有节点确保自己是最小的节点呢？ 假设某个client在获得锁之前挂掉了, 由于client创建的节点是ephemeral类型的, 因此这个节点也会被删除, 从而导致排在这个client之后的client提前获得了锁. 此时会存在多个client同时访问共享资源.
如何解决这个问题呢? 可以在接到waitPath的删除通知的时候, 进行一次确认, 确认当前的thisPath是否真的是列表中最小的节点.
结论
上次两个分布锁实现，都是可行的。具体选择哪个，取决于你的集群规模。
```

###FAQ

1. 客户端对ServerList的轮询机制是什么
随机，客户端在初始化( new ZooKeeper(String connectString, int sessionTimeout, Watcher watcher) )的过程中，将所有Server保存在一个List中，然后随机打散，形成一个环。之后从0号位开始一个一个使用。
两个注意点：1. Server地址能够重复配置，这样能够弥补客户端无法设置Server权重的缺陷，但是也会加大风险。（比如: 192.168.1.1:2181,192.168.1.1:2181,192.168.1.2:2181). 2. 如果客户端在进行Server切换过程中耗时过长，那么将会收到SESSION_EXPIRED. 这也是上面第1点中的加大风险之处。更多关于客户端地址列表相关的，请查看文章《ZooKeeper客户端地址列表的随机原理》

2.客户端如何正确处理CONNECTIONLOSS(连接断开) 和 SESSIONEXPIRED(Session 过期)两类连接异常
在ZooKeeper中，服务器和客户端之间维持的是一个长连接，在 SESSION_TIMEOUT 时间内，服务器会确定客户端是否正常连接(客户端会定时向服务器发送heart_beat),服务器重置下次SESSION_TIMEOUT时间。因此，在正常情况下，Session一直有效，并且zk集群所有机器上都保存这个Session信息。在出现问题情况下，客户端与服务器之间连接断了（客户端所连接的那台zk机器挂了，或是其它原因的网络闪断），这个时候客户端会主动在地址列表（初始化的时候传入构造方法的那个参数connectString）中选择新的地址进行连接。
好了，上面基本就是服务器与客户端之间维持长连接的过程了。在这个过程中，用户可能会看到两类异常CONNECTIONLOSS(连接断开) 和SESSIONEXPIRED(Session 过期)。
CONNECTIONLOSS发生在上面红色文字部分，应用在进行操作A时，发生了CONNECTIONLOSS，此时用户不需要关心我的会话是否可用，应用所要做的就是等待客户端帮我们自动连接上新的zk机器，一旦成功连接上新的zk机器后，确认刚刚的操作A是否执行成功了。
SESSIONEXPIRED发生在上面蓝色文字部分，这个通常是zk客户端与服务器的连接断了，试图连接上新的zk机器，这个过程如果耗时过长，超过 SESSION_TIMEOUT 后还没有成功连接上服务器，那么服务器认为这个session已经结束了（服务器无法确认是因为其它异常原因还是客户端主动结束会话），开始清除和这个会话有关的信息，包括这个会话创建的临时节点和注册的Watcher。在这之后，客户端重新连接上了服务器在，但是很不幸，服务器会告诉客户端SESSIONEXPIRED。此时客户端要做的事情就看应用的复杂情况了，总之，要重新实例zookeeper对象，重新操作所有临时数据（包括临时节点和注册Watcher）。

3. 不同的客户端对同一个节点是否能获取相同的数据

4. 一个客户端修改了某个节点的数据，其它客户端能够马上获取到这个最新数据吗

ZooKeeper不能确保任何客户端能够获取（即Read Request）到一样的数据，除非客户端自己要求：方法是客户端在获取数据之前调用org.apache.zookeeper.AsyncCallback.VoidCallback, java.lang.Object) sync.
通常情况下（这里所说的通常情况满足：1. 对获取的数据是否是最新版本不敏感，2. 一个客户端修改了数据，其它客户端需要不需要立即能够获取最新），可以不关心这点。
在其它情况下，最清晰的场景是这样：ZK客户端A对 /my_test 的内容从 v1->v2, 但是ZK客户端B对 /my_test 的内容获取，依然得到的是 v1. 请注意，这个是实际存在的现象，当然延时很短。解决的方法是客户端B先调用 sync(), 再调用 getData().

5. ZK为什么不提供一个永久性的Watcher注册机制

不支持用持久Watcher的原因很简单，ZK无法保证性能。
6. 使用watch需要注意的几点

a. Watches通知是一次性的，必须重复注册.
b. 发生CONNECTIONLOSS之后，只要在session_timeout之内再次连接上（即不发生SESSIONEXPIRED），那么这个连接注册的watches依然在。
c. 节点数据的版本变化会触发NodeDataChanged，注意，这里特意说明了是版本变化。存在这样的情况，只要成功执行了setData()方法，无论内容是否和之前一致，都会触发NodeDataChanged。
d. 对某个节点注册了watch，但是节点被删除了，那么注册在这个节点上的watches都会被移除。
e. 同一个zk客户端对某一个节点注册相同的watch，只会收到一次通知。即
```java
for( int i = 0; i < 3; i++ ){
    zk.getData( path, true, null );
    zk.getChildren( path, true );
}
```
f. Watcher对象只会保存在客户端，不会传递到服务端。
7.我能否收到每次节点变化的通知

如果节点数据的更新频率很高的话，不能。
原因在于：当一次数据修改，通知客户端，客户端再次注册watch，在这个过程中，可能数据已经发生了许多次数据修改，因此，千万不要做这样的测试：”数据被修改了n次，一定会收到n次通知”来测试server是否正常工作。（我曾经就做过这样的傻事，发现Server一直工作不正常？其实不是）。即使你使用了GitHub上这个客户端也一样。

8.能为临时节点创建子节点吗

不能。

9. 是否可以拒绝单个IP对ZK的访问,操作

ZK本身不提供这样的功能，它仅仅提供了对单个IP的连接数的限制。你可以通过修改iptables来实现对单个ip的限制，当然，你也可以通过这样的方式来解决。https://issues.apache.org/jira/browse/ZOOKEEPER-1320

10. 在getChildren(String path, boolean watch)是注册了对节点子节点的变化，那么子节点的子节点变化能通知吗

不能

11.创建的临时节点什么时候会被删除，是连接一断就删除吗？延时是多少？

连接断了之后，ZK不会马上移除临时数据，只有当SESSIONEXPIRED之后，才会把这个会话建立的临时数据移除。因此，用户需要谨慎设置Session_TimeOut

12. zookeeper是否支持动态进行机器扩容？如果目前不支持，那么要如何扩容呢？

截止2012-03-15，3.4.3版本的zookeeper，还不支持这个功能，在3.5.0版本开始，支持动态加机器了，期待下吧: https://issues.apache.org/jira/browse/ZOOKEEPER-107

13. ZooKeeper集群中个服务器之间是怎样通信的？

Leader服务器会和每一个Follower/Observer服务器都建立TCP连接，同时为每个F/O都创建一个叫做LearnerHandler的实体。LearnerHandler主要负责Leader和F/O之间的网络通讯，包括数据同步，请求转发和Proposal提议的投票等。Leader服务器保存了所有F/O的LearnerHandler。

14.zookeeper是否会自动进行日志清理？如果进行日志清理？

zk自己不会进行日志清理，需要运维人员进行日志清理，详细关于zk的日志清理，可以查看《ZooKeeper日志清理》