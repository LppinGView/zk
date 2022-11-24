# zookeeper学习笔记


## 一、关于Zookeeper Atomic Broadcast zab协议

### [totally-ordered-broadcast-protocol](http://diyhpl.us/~bryan/papers2/distributed/distributed-systems/zab.totally-ordered-broadcast-protocol.2008.pdf)

### 1.介绍
 * zookeeper高可用一致性服务,一般用于读写比例为2:1的场景.读可以在当前服务中获取一致性视图,而写需要重定向到leader中,进行生成事务号,进行2pc提交,等待大多数节点同步成功,则事务成功,并返回结果.
 * 写过程: 写请求重定向到leader -> 生成transaction事务号,原子广播delive -> 从节点收到,放置到历史队列中,并返回主节点ack -> 主节点收到大多数ack(大于一半以上法定成员数),提交commit -> 从节点收到commit,依次提交历史队列中,待提交事务.


### 2.必要条件
 + reliable delivery 可靠性投递: 消息m被一个服务投递,则最终被所有法定成员投递;
 + total order 整体有序: 消息a在一个服务中,在消息b之前被投递,则在每个服务中投递b之前,一定会先投递a;
 + causal order 因果有序: 如果由于消息a产生了消息b,且这两个消息都要被投递,则消息a必须先于b进行投递;
 + prefix property 前置条件: 如果消息m是 leader节点L 的最新的投递消息,则任何在m之前的所有消息,必须都已经被L投递;

#### 基于以上的必要条件,就可以构建正确的zookeeper数据库副本
	条件1和条件2 保证 所有的副本具有一致性视图;  
	为啥? 1保证节点互通 2保证整体消息有序
	
	条件3 保证 副本对于使用zab协议的应用来说有一个正确的状态;

	leader 基于接收到的请求去更新数据库。

#### 另外有两个因果关系的讨论
 * 消息a和消息b 被同一个服务发送,并且消息a 在消息b之前 发起提议, 则a是b的因;
 * zab假定,任何时候都只有一个leader提交议题.如果一个leader改变,任何之前leader的commit提议都是新leader commit提议消息的因;

#### 因果违反事例 -违反因果关系导致的问题,从另外角度认识因果关系
 * 设想如下:
   * 客户端C1 通过消息w1 请求 set一个节点 /a 的值为1   请求参数为("/a", 1, 1)  (/zode, value, version)
   * 客户端C1又 通过消息w2 请求 set 节点/a 的值为2 请求参数("/a", 2, 2)
   * 服务端leader L1 提议并发出w1, 之后发送w2(本地已保存), 但是发往其他节点时失败了

 * 一个新的leader L2 接管
   * 客户端C2 请求set节点/a为3, 基于版本号1, 它将发起消息w3,格式为("/a", 3, 2)
   * L2 提议并投递w3
	
#### 以上设想,w1请求成功.最终L1故障恢复,并被选举为leader,尝试提议w2,并投递它时,将会导致违反因果关系,并导致副本集状态错误


### 3.失败模型
  * zab的失败模型是基于状态恢复的故障失败(系统可以自恢复,而不是机器宕机等故障).各服务对时间的敏感程度大致相同(通过超时时间进行探测失败).
组成zab的进程,具有持久化状态,当故障恢复时,能通过持久化状态进行恢复.这意味着,一个进程的状态可能是部分有效的,如果丢失了一些最近的事务trancation,或者更严重的是,存在transaction但是没有投递,这些对于该进程来说必须跳过!(这可以避免上述违反因果的情况).
	

  * 我们可以处理n台机器的故障,同时也要求可以处理相关的可恢复故障,如断电.这要求我们投递过得消息,必须存在相应法定成员的磁盘中.


  * 尽管我们不承担解决拜占庭失败,但是我们使用摘要进行数据错误检测.通过在协议包中添加额外的元数据,进行异常检测.如果检测到数据错误或者异常包,将会丢弃.


  * 操作环境和协议本身是独立实现的,这就导致,实现一个完全的拜占庭容忍系统对我们的应用来说是不切实际的.这也让我们看到,实现一个这样的正确的独立系统光靠编程资源是不够的.


  * zookeeper使用内存数据库,将事务日志和周期快照存储在磁盘磁盘中(顺序写).zookeeper的事务日志兼做数据库的写前日志,故一个事务只需一次落盘.zookeeper基于内存,配上千兆网卡,使得磁盘的写I/成为了瓶颈.为了缓解磁盘I/O写瓶颈,通过批处理,将多次事务通过一次写操作记录到磁盘,同时批处理发生在各副本内部,不在协议层面,所以它的实现,同消息打包相比更多的是和组提交有关.我们不使用消息打包来减少延迟,但是通过批处理I/O打包到磁盘,我们仍然获得的大部分好处.


  * 当一个服务恢复时,它将读取它的快照,同时在快照之后重放所有已经投递的事务.因此,在恢复阶段,原子广播协议不需要保证至多一次投递.我们的事务幂等设计,保证了,只要重新开始的顺序正确,多次投递同个事务也是ok的.这是一个广义上的总体有序要求(上述条件2).特别的,当一个事务a在事务b之前已经投递,当失败之后,事务a重新投递,b总是在a之后才进行投递.


### 4.其他性能要求
  * low latency 低延时: zookeeper 被应用广泛使用，用户希望请求低延时；
  * bursty high throughput 间歇性高吞吐量: 应用使用zookeeper通常是读比重高的工作场景，但是偶尔彻底的重配置发生时，会导致一个大的写吞吐量高峰。
  * smooth failure handling 平滑的失败处理: 如果一个不是leader的服务挂了，当前集群还有正确的服务，那么整个服务功能应不受影响。


### 5.为什么需要一个新的协议
#### 已知的协议
 * ABCAST： total order
 * CBCAST： causal order
 * paxos： 有一系列重要的属性，在实际假设中，当一部分进程失败，它允许进程崩溃、恢复，并且能够保证commit操作在3个交流步骤中完成；

#### 通过两个实际假设，我们简化paxos，可以获得高吞吐量。
 * 1.paxos容忍消息丢失和重置消息顺序。各服务间通过使用TCP进行交流，使消息按照FIFO的顺序进行投递，这能保证每个提议具有因果顺序，即使服务进程存在未完成提议消息。然而，paxos不能保证因果顺序，因为它没有要求FIFO channels;

 * 2.在paxos中提议者是为不同服务实例进行提议的代理。为了保证议程能推进，每次必须是单个提议者进行提议。否则，提议者永远会在一个给定的实例中进行竞争，这样一个有效的提议者就是一个leader。当paxos从主节点失败中恢复，新的leader确保所有的已投递的消息进行了投递，同时重新发起之前leader遗留的提议。

#### 因为多个leader能够对一个实例进行提议，这就产生了两个问题。
 * 提议会产生冲突；paxos使用投票机制，去检测和解决冲突提议。
 * 不能知道一个提议是否已经提交，进程必须能够辨别出提议是否提交。

#### zab通过当前仅仅只有一个提议消息会发出，从而避免以上两个问题。这避免了投票机制，并简化了恢复过程。
#### 在paxos中，如果一个服务相信自己是leader，需要通过高选票从之前的leader手中夺取领导权。而，zab中，一个新leader，可以通过法定成员抛弃之前的leader或者之前leader服务异常时，获取领导权。
#### 另一个获取高可用的可选方式为：限制每个广播消息协议的复杂性，例如使用固定序列器环FSR来广播消息。使用FSR，当服务在增加时，吞吐量不会降低，但是延时会随着进程数增加而增加，这不利于我们应用。当系统足够的稳定，虚拟同步也能有高吞吐量。然而任何服务的失败，如果导致服务重新配置，那么在重新配置的过程中，系统将短暂中止服务。而且在这样的系统中，一个失败检测器需要监测所有服务。一个稳健的失败检测器，对于重新配置至关重要。


### 6.新协议-zab
zab分为recovery 和 brocaster
未完待续...


## 二、zookeeper源码分析（未完待续）

### 源码编译运行
* 跳过测试编译 mvn -DskipTests=true clean package
* 复制配置文件 cp zoo_sample.conf zoo.cfg
* 编辑服务端启动参数 QuorumPeerMain，配置：Program arguments：conf/zoo.cfg，注释：相关jetty scope
* 启动根据error，处理错误
* 编辑客户端启动参数 ZooKeeperMain，配置：Program arguments：-server 127.0.0.1:端口，注释：commons-cli scope


### 1.FastLeaderElection 使用tcp进行服务间选举，构造函数有两个参数(当期成员对象QuorumPeer，连接管理器QuorumCnxManager)
 * Notification 通知，用来通知其他成员，它自己改变了投票  由于加入选举或者它了解到其他服务具有更高的zxid或者相同的zxid但是有更高的server id
ToSend 消息，一个成员发往另一个成员
 * Messenger 多线程实现消息的处理，构造函数传入QuorumCnxManager
	* WorkReceiver 使用一个新线程, 从管理者实例(QuorumCnxManager)的run方法中接收到消息，然后处理该消息
	* WorkSender 使用一个新线程，将发送消息出队？并放入管理者的队列(QuorumCnxManager)
	* ZooKeeperThread zookeeper封装的线程
 * SyncedLearnerTracker 包含一个选票集合，该对象用于决定是否有充足的选票宣布本次选举结束

 * zxid 64bit 前32位表示epoch 后32位为提议的序号，epoch每次选举结束增加，提议序号重置，当前选举期内，提议序号自增。
 * a peer 可能是参与者或者是观察者
 * LearnerType 这个用于决定当共识改变时，服务的状态走向
	* PARTICIPANT 参与者：参与选举或者成为leader的共识中进行投票
	* OBSERVER 观察者：不参与选举，且不会被选举为leader（负责客户端查询请求等非事务性的会话请求）

 * 只有leader和follower的集群 cp （缺点）：
	* 1.随着集群规模的变大，集群处理写入的能力下降；
	如果当前集群follower越多，一次性写入投票等相关操作就约复杂，并且follower服务器间通讯变得越来越耗时，事务性处理性能就越低；
	故，3.6之后，集群引入了新的服务角色，Observer，它可以接收非事务性请求，并且不参与投票的选举操作；（不参与投票，只能通过和主节点间的同步，进行数据一致性处理。所以会有数据不一致情况）
	这样既保证了，集群性能的扩展。
    * 2.zookeeper集群无法做到跨域部署；（由于需要经常进行网络通讯，需要将leader和follower配置在同一个网络中。网络分区，将影响到服务重新进行选举）
	在实际部署中，因为Observer不参与Leader的选举，并不会向Follower服务器那样频繁与Leader进行通讯。因此可以将Observer部署到不同的网络分区中，这样也不会影响整个Zookeeper集群的能力，也就是所谓的跨域部署。
 * ServerState 服务状态
  
		LOOKING
		FOLLOWING
		LEADING
		OBSERVING
 * ZabState 协议状态，用于监控，显示当前运行的成员的zab协议状态
	
		ELECTION
		DISCOVERY
		SYNCHRONIZATION
		BROADCAST

### 2.QuorumPeer：用于管理法定成员协议。服务有三个状态：
 * Leader election： 服务间选举leader，每个服务初始化选举自己
 * Follower： 服务将和leader进行同步，并复制任何事务
 * Leader： 该leader服务将处理请求，并转发给followers，一半以上成员日志记录该请求后，leader才能接收该请求。
   * Map<Long, QuorumPeer.QuorumServer> view 每个服务对全局服务的视图
 * QuorumBean 成员bean,包含成员属性

		LocalPeerBean
		RemotePeerBean
		LeaderElectionBean
 * JvmPauseMonitor 监控jvm或者虚拟机暂停时间
 * AddressTuple 地址数组(MultipleAddresses、InetSocketAddress)
 * QuorumServer 成员服务实体

### 3.QuorumCnxManager 服务连接管理器
 * Message 消息体
 * InitialMessage 初始消息体
 * ThreadGroup 线程组
 * ThreadFactory 线程工厂
 * ThreadPoolExecutor
 * QuorumCnxManager构造函数 
   
		初始化连接
 		initializeConnectionExecutor


### 4.个基础
#### zookeeper-jute 序列化和反序列化
 * Record 记录操作
 * Index 迭代器
 * OutputArchive 输出对象
 * InputArchive 输入对象

#### 持久化 zookeeper-server src/main/java/org.apache.zookeeper
 * TxnLog 日志操作 wal写前日志技术，保证crash安全
 * SnapShot 快照操作，用于将内存快照保存磁盘
 * FileTxnLog 日志操作实现类
 * FileSnap 快照操作实现类
 * FileTxnSnapLog 日志和快照的helper类 内部类有TxnLog和SnapShot

#### 网络传输
 * ServerCnxnFactory 用于创建ServerCnxn的工厂 org.apache.zookeeper.server.ServerCnxnFactory
 * ServerCnxn 用于监听zoo.cfg配置中的clientPort端口，当客户端发起与服务端连接，服务器将创建一个ServerCnxn实例，用于该连接。系统默认为NIOServerCnxn
 * ClientCnxn 客户端连接类，用于管理有效的服务端连接列表

#### 监听机制（观察者模式？）
 * WatchManager 服务端用于管理Watcher、触发器等 
	* triggerWatch 触发方法，会对watcher封装相应的WatchedEvent，然后通过ServerCnxn.process(WatchedEvent)进行处理，实际是通过NIOServerCnxn.sendResponse进行发送。
	* 发送过程：
		* 将数据放入到队列
		* ServerCnxnFactory，在获取到可写事件时，默认使用NIOServerCnxnFactory，将队列中的数据发送出去。WorkerService.schedule,使用线程池将写操作进行处理
 * Watcher 监听器
	* Event 监听器事件描述类
		* KeeperState 连接状态枚举，用于表示客户端与服务端连接状态
		* EventType 事件类型枚举
	* WatcherType 枚举类，标识监听器的类型
	* WatchedEvent 监听事件（包含，连接状态keeperState、事件类型EventType、节点路径path）

#### Zookeeper 客户端API获取连接之后的操作入口类
 * WatchRegistration 在命令发送时注册监听器，之后通过连接向服务端发送该监听器;
 * ClientWatchManager materialize物化方法 客户端从服务端接收已被触发的监听器，从而处理后置处理事件

#### 监听器总结：
 * 客户端：监听器注册（客户端设置监听器，同时配置监听触发事件，ClientCnxn通过网络发送到服务端，ServerCnxn服务端配置监听。）;
 * 服务端：监听器触发（相应操作触发监听，通知客户端，客户端收到事件，进行后置处理）

#### 其他相关基础
 * 并发相关、线程池等等
 * 文件IO操作相关
 * 网络编程NIO流程

#### 源码跟踪启动
 * QuorumPeerMain.main方法
   * 1.读取配置文件，装配成QuorumPeerConfig
   * 2.开启磁盘快照清除任务DatadirCleanupManager.start(),使用Timer进行调度，PurgeTxnLog.purge
   * 3.创建服务连接工厂ServerCnxnFactory.createFactory()  环境变量中没有设置zookeeper.serverCnxnFactory参数，就默认使用NIOServerCnxnFactory
   * 4.ServerCnxnFactory.configure(),启动ServerSocketChannel.open, 选择器打开，注册OP_ACCEPT事件 
	 * 配置选择器线程数、工作线程数，获取选择器线程迭代器
   * 5.创建QuorumPeer：用于管理成员协议,服务器状态可能为：leader 选举、follower、leader，同时设置了一个socket,总是返回当前leader的视图(xid, myid, leader_id, leader_zxid)
   * 6.QuorumPeer.start()
		* 装载zk数据库ZKDatabase.loadDataBase() 从快照文件中恢复数据库 FileTxnSnapLog.()
		* 开始服务端连接工厂：QuorumPeer.startServerCnxnFactory()， NIOServerCnxnFactory.start(),创建工作线程池new WorkerService，创建多个选择器线程 SelectorThread，创建AcceptThread
		  * startLeaderElection开始选举
			* 选举开始，通过UDP广播自己的选票 new Vote()
		* QuorumCnxManager 负责选举过程中的网络通讯  3888 使用Listener进行处理(Receiver， Sender).start -> run方法 启动了两个线程
		* ServerCnxnFactory 负责客户端与服务端之间的网络通讯 2181
		* FastLeaderElection 中的会有两个线程（选票toSend， 我收到的投票message）
	
 * QuorumCnxManager 负责节点间选举通讯        

						接收外部选票                               发送出去
							|                                     /|\
						       \|/				sendWorker      sendQueue
						    RecvWorker                          listener --> sendWorker      sendQueue
							|                               sendWorker      sendQueue
							|负责接收message，然后放入        sendWorkerMap   queueSendMap
						       \|/
				——————————————————————————————————————————————————————
			       |                 recvQueue<Message>                   |
				——————————————————————————————————————————————————————       
						  |			         /|\
						  |recvQueue.poll 	          |
						  |				  |
				_________\|/___________				  |
			       |recvqueue<Notification>|			  |manager.toSend recvQueue.add(msg)
				———————————————————————                           |
						|  /|\				  |
                  manager.pollRecvQueue|   |			      WorkerSender <—————————————|
						|   |						         |
						|   |recvQueue.offer()				         |sendqueue.poll
						|   |                    			         |
					       \|/  |							 |        
					WorkerReceiver   --------> sendqueue.offer(msg) ----> sendqueue<ToSend>
   
 * FastLeaderElection

#### 选举机制
 * 三个条件
	* 1.epoch
	* 2.zxid
	* 3.serverid = myid
 * 选票：发送给其他人的提案，其他人可以赞成或者发对
 * 投票：对选票投 赞成或者反对
   * 1.首先选举epoch大的，一切epoch小的都不是我们的选择
   * 2.当epoch一样时，选zxid大的
   * 3.当zxid一样，选serverid大的

#### 总的原则：
 * 1.每个server一启动（一上线都是LOOKING状态），都是把自己生成的 epoch zxid myid 构建成自己的选票，广播给所有其他节点（通俗的说：每个server一启动就去找leader，因为不确定leader是否存在，所以在第一次找leader的时候，就会构建自己的选票，然后广播给其他server）;
 * 2.每次收到一个投票，就先进行一次关于 epoch zxid myid 的比较，如果自己的不满足要求，就把自己的选票更换成别人的，然后再次广播；
 * 3.如果每次接收到的投票中，有一个投票是赞成自己的，那么执行判断方法，来判断，迄今为止接收到的所有投票，是否有超过半数，是同意自己的；
 * 4.如果有，那么就更改自己的状态(LOOKING变成LEADING）因为有可能会出现，其他的server还在投票，而且含有大于自己的epoch的。把自己的选票在广播一次；
 * 5.再获取一个投票，看是否获取到的投票的epoch zxid myid 是否比自己大，如果不比自己大，则成为leader。
	
		1         3  
		   
		     2

 * 1.上线先广播自己选票（未完待续..）
 * 2.收到投票，进行唱票和自己的比较 < 
	* 把自己的选票换成别人的，再次广播 否则，把自己

 
 * NIOServerCnxnFactory
   * 1.一个接收线程 将接收到的
	
#### 核心源码（未完待续...）
#### 启动
#### 选举
#### 副本间同步(恢复、广播)
 * 1.zk中的队列有哪些？
 * 2.zk中的网络通讯包格式有哪些？
	INFORM\Proposal
 * 3.服务端接收到请求后，怎么处理？（使用处理链进行处理）？
 * 4.一个运行中的zookeeper服务会产生哪些数据和文件？（从源码角度看？）
	* 从存放位置看：
		* 内存数据
	    * 磁盘数据
	* 从种类和作用来看？：
		* 事务日志数据（当前请求时，DataTree会更新同时记录事务日志，事务日志用于节点间数据同步）
		* 数据快照数据（用户节点故障恢复时，服务重新构建DataTree，磁盘中的事务日志 + 数据快照）

## balabala

=======================================

				---------------------------------------
				---///////////////-------////////------
				-------- / ------------///-----   -----
				-------- / -----------///-------   ----
				-------- / ----------///---------   ---
				-------- / ---------///-----------   --
				-------- / ---------///-----------   --
				-------- / ---------///-----------   --
				-------- / ----------///---------   ---
				-------- / -----------///-------   ----
				-------- / ------------///-----   -----
				---///////////////------/////////------
				---------------------------------------
=======================================

### I/O NIO
 * 用户态进程如何调用系统调用?
 * 用户态进程通过调用系统调用 system call向内核发送指令完成操作
#### 同步阻塞IO

	用户进程     ----system call------       内核进行
											  	
	 					等待可读(这期间做了什么? 参考操作系统)

						可读
	
						内核将内核空间数据copy到用户空间
	
		     <-----读返回----            读完成
#### 应对策略:
	多线程 -> 线程池
 * 优点:线程挂起,不会占用cpu时间
 * 缺点:高并发下,需要大量的线程来维护大量的网络连接,内存\线程切换开销非常巨大,高并发场景下性能很低,基本不可用

#### 同步非阻塞IO (socket设置为非阻塞方式)
 * 优点:线程挂起,不会占用cpu时间
 * 缺点:高并发下,需要线程不停轮询内核,占用大量cpu时间,系统消耗大

#### 问题
 * 怎么从IO模型
 * java io > javaNIO ?
 * 高级nio模型 reactor
 * 框架netty

#### java 线程
 * 线程模型
	* 应用线程 1:1内核线程
	* 用户线程 N:1内核线程
	* 混合线程模型 用户线程N:M内核线程
 * 根据线程的行为方式又分为:
	* 抢占式线程: 线程启动后(start), 何时被cpu执行,执行多长时间,不受线程自己控制,交由操作系统
	* 协同式线程: 用户线程能够主动控制线程的执行过程,执行多久,多长时间等等.不由操作系统控制
 * java线程模型属于: 
	* 应用线程 1:1内核线程
	* 抢占式线程模型
	
#### 面对高并发情况,有很多弊端（说的啥...）
 * 中断,保护,切换上下文等 
 * 线程的创建消耗,很消耗资源 
 * 线程的上下文切换,耗时

#### 线程池
 * 目前的应对策略,使用线程池进行线程管理

#### kotlin
 * 基于jvm实现的其他语言生态,实现了用户线程模型,如kotlin,但是也有问题(什么问题?)

#### go
 * 其他语言

#### 纤程
 * java 未来的应对策略
 * Project Loom项目是通过优化jvm传统线程模型,优化成纤程(轻量级用户线程)实现的.从jvm上支持N多的用户线程,也无需管理线程池.

### java 线程生命周期

	创建(New)     /超时等待(Time_wait)
		\       /
        start\     /sleep(10)\wait(10)\join(10)\park(10)
		  \   /
		  运行(Running)----wait/park/join---notify\notifyAll---无限等待(Wait)
		 /   \
        sync/     \
	       /       \run结束
	      /         \
     阻塞(Blocked)  死亡(Termination)

 * Object.wait/LockSupport.park/Thread.join
   * sleep 不会释放持有的锁等资源
   * wait 会释放持有的锁等资源
   * 无限等待: 不会被分配cpu时间,进入等待状态,需要被其他进程唤醒
   * 超时等待: 不会被分配cpu时间,进入超时等待状态, 不需要被其他线程唤醒,到时它们会被系统自动唤醒
   * 阻塞: 线程进入阻塞状态,不会被分配cpu时间,当线程进入同步区域时,线程处于阻塞状态.阻塞状态的线程在等待着获取一个排他锁,这个事件将在另一个线程放弃这个锁时发生.
	
 * park是基于什么实现的?
 * join是基于什么实现的?

#### java 并发编程
 * 为啥需要并发编程?
 * 从单核到多核?说明了,单靠一个机器处理事务,有诸多问题(单点\不好弹性扩容\等等)
 * 系统架构也从单体走向了分布式处理

#### 那并发编程有啥问题吗?(有序性\原子性\可见性)
 * 可以从硬件系统中获取对应方法去解决吗?
 * java提出了什么样的内存模型?
 * 针对不同的并发问题,java内存模型是如何处理这些问题的?
 * juc类

#### 课程作业（balabala...）
 * 如何使用go实现一个轻量级zookeeper?