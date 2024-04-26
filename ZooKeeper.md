#    应用场景

提供的服务包括：统一命名服务，统一配置管理，统一集群管理，服务器节点动态上下线，软负载均衡等。



**统一配置管理**

1. 配置文件同步；
2. 对配置文件修改后，希望能够快速同步到各个节点上；
3. 各个客户端监听这个 Znode；

<img src="https://cdn.jsdelivr.net/gh/StuartYang/oss@master/img/202401130211425.png" alt="image-20240112214449530" style="zoom:50%;" />

**统一集群管理**

1. 可将节点信息写入 ZK 上的一个 ZNode
2. 监听这个 Znode 可获取它的实时状态变化



**服务器动态上下线**

服务器注册到 ZK 上，客户端监听 ZNode 上的信息，动态//监听服务信息。



# ZK 命令

> Docker安装：https://blog.csdn.net/lanse_huanxiang/article/details/129431212

集群安装

1. 主目录在创建 `zkData` 目录

2. `zkData`目录里创建 `myid` 文件

3. 在 `myid` 文件中添加server 对应的编号（相当于该台 zk 的身份表示）：1,2,3

4. 修改 `zoo.conf `的数据存储路径

5. 在 `zoo.conf `的增加如下信息

   ```shell
   server.1 = ip1:2888:3888
   server.2 = ip2:2888:3888
   server.3 = ip3:2888:3888
   
   serer.A = B:C:D
   ##
   A: 一个编号，代表第几台服务器
   B: 该服务器地址
   C：follower 和 leader交换信息端口
   D：如果 leader 宕机，选举端口
   ##
   ```



# ZK选举

zk 中的 ID 类型：

1. SID：服务器 ID，和 myid 一致；
2. ZXID：事务 ID，用来标识一次客户端对服务状态的更改；
3. Epoch：任期编号，每投完一次票会增加；



## 第一次启动

1. 服务器 1 启动，发起一次选举，服务器 1 投自己一票，不够半数以上（3 票），选举无法完成，状态保持 looking；
2. 服务器 2 启动，再一次选举，服务器 1 和 2 分别投自己一票减缓选票信息，此时服务器 1 发现 2 的 myid 比自己大，更改为推举服务器 2，此书服务器 1 为 0 票，服务器 2 票，没有达到半数，服务器 1,2 依旧是 looking 状态；
3. 服务器 3 启动，发起投票，服务器 1和 2 改选票为 3，这是服务器 1 和 2 为 0 票，服务器 3 为 3 票，此时服务器 3 的票数已经超过半数，服务 3 当选 leader，服务器 1 和 2 状态为 following，服务器 3 状态为 leading；
4. 服务器 4 启动.发起一次选举，此时服务器 1 和 2 已经不是 looking 状态，不会更改选票信息。交换选票信息结果：服务器 3 为 3 票，服务器 4 为 1 票，此书服务器 4 服从多数，更改选票信息为服务器 3，并更改状态为 following；
5. 服务器 5，同 4；



## 非第一次启动



**服务器运行期间无法和 leader 保持连接时**

当一台机器进入 leader 选举流程时候，当前几圈也可能处于两种状态：

1. 存在 leader：机器尝试选举 leader 时，会被告知 leader 信息，对于改机器来说，仅仅需要和 leader 机器简历连接，并同步状态信息。
2. leader 宕机
   1. 假设 zk 由5 台机器组成，SID 分别是 1,2,3,4,5，ZXID 分别是 8,8,7,7，,并且此时 SID 为 3 的服务器是 leader ，某一时刻，3 和 5 出现故障，开始进行 leader 选举。
   2. 选举规则：（Epoch，ZXID，SID）
      1. Epoch 大的直接胜出
      2. EPOCH 相同，事务 ID 大的胜出
      3. 事务 ID 相同，服务器 ID大的胜出



# ZK命令行常用操作

```shell
  # 打开id上的客户端
  bin/zkCli.sh -server id:2181
  
  ls -s /  # 根节点信息
```



## 节点类型

- 持久节点：客户端和服务断开连接后，创建的节点不删除
  - 不带序号：例如目录节点`zode1`
  - 带序号：例如目录节点`zode1_001`，顺序号在分布式系统中可以用于为所有的事件进行全局排序，这样客户端可以通过序号推断事件的顺序。
- 短暂节点：断开后，创建的节点会自己删除
  - 带序号
  - 不带序号

```shell
  # 永久不带序号节点
  create /节点名 "内容"
  # 获取节点信息
  get -s /节点名
  # 带序号永久
  create -s /节点名 "内容"
  # 带序号短暂
  create -e -s /节点名 "内容"
```



# 监听器

>  监听器保证了zk 客户端可以及时拿到服务端的信息。



![未命名绘图.drawio](https://cdn.jsdelivr.net/gh/StuartYang/oss@master/img/202401130212392.png)

常见的监听：

1. 监听节点的数据变化
2. 监听子节点增减的变化



```shell
# 监听节点变化，注意注册一次，生效一次
get -w /目录节点名
# 节点的子节点变化（路径变化）
ls -w /目录节点名
```



# API操作

依赖

```xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.9</version>
    <type>pom</type>
</dependency>
```



**API**

创建客户端

- connectString :  指zk的服务器列表，以英文输入法下逗号分割的host:port；例如：192.168.2.14:2181,192.168.1.15:2181 
- sessionTimeout : 会话超时时间，单位是毫秒 。当在这个时间内没有收到心跳检测，会话就会失效 ；
- watcher : 注册的watcher，null表示不设置 ；
- canBeReadOnly : 用于标识当前会话是否支持"read-only"模式 ；
- sessionId : 会话ID 
- sessionPasswd : 会话密钥

```java
1) ZooKeeper(String connectString, int sessionTimeout, Watcher watcher)
2) ZooKeeper(String connectString, int sessionTimeout, Watcher watcher, boolean canBeReadOnly)
3) ZooKeeper(String connectString, int sessionTimeout, Watcher watcher, long sessionId, byte[] sessionPasswd)
4) ZooKeeper(String connectString, int sessionTimeout, Watcher watcher, long sessionId, byte[] sessionPasswd, boolean canBeReadOnly)
```



创建节点

- path:被创建的节点路，比如：/zoo/tiger 

- data[]:节点数据 acl:Acl策略 
- createMode: 节点类型，枚举类型，有四种选择:
  - 持久 PERSISTEN    
  - 持久顺序 PERSISTENT_SEQUENTIAL    
  - 临时 EPHEMERAL    
  - 临时顺序 EPHEMERAL_SEQUENTIAL 
- cb : 异步回调函数，需要实现StringCallback接口，当服务端创建完成后，客户端会自动调用这个对象的processResult方法 
- ctx : 用于传递一个对象，可以在回调方法执行的时候用，通常用于传递业务的上下文信息 

```java
#同步创建节点
1) String create(String path, byte[] data, List<ACL> acl, CreateMode createMode) 
#异步创建节点
2) void create(String path, byte[] data,List<ACL> acl, CreateMode createMode, AsyncCallback.StringCallback cb, Object ctx); 

以上创建方法都不支持递归创建节点，当节点存在时抛出异常NodeExistsException

 
```





删除节点

- path:被删除的节点路径 
- version:知道节点的数据版本，如果指定的版本不是最新版本，将会报错，它的作用类似于hibernate中的乐观锁
-  cb:异步回调函数 
- ctx:上下文信息 

```java
 #同步删除节点
1) void delete(String path, int version); 
#异步删除节点
2) void delete(String path, int version, AsyncCallback.VoidCallback cb, Object ctx); 
```



获取子节点

- path:数据节点路径
-  watcher:设置watcher后，如果path对应节点的数据发生变化，将会得到通知，允许为null 
- watch:是否使用默认的watcher 
- stat:指定数据节点的状态信息 
- cb:异步回调函数 
- ctx:上下文信息 

```java
void getChildren(String path, Watcher watcher, AsyncCallback.ChildrenCallback cb, Object ctx)
```



获取节点数据

```java
void getData(String path, Watcher watcher, AsyncCallback.DataCallback cb, Object ctx)
byte[] getData(String path, Watcher watcher, Stat stat)
```



修改节点数据

```java
Stat setData(String path, byte[] data, int version)
void setData(String path, byte[] data, int version, AsyncCallback.StatCallback cb, Object ctx)
```



检测节点是否存在

```java
Stat exists(String path, boolean watch)
void exists(String path, boolean watch, AsyncCallback.StatCallback cb, Object ctx)
Stat exists(String path, Watcher watcher)
```



案例

```java
public class ZookeeperTest {

    private static final String URL = "127.0.0.1:2181";

    private ZooKeeper zk;

    @Before
    public void setUp() throws Exception {
        zk = new ZooKeeper(URL, 20000, new Watcher() {
            // 监控所有被触发的事件
            @Override
            public void process(WatchedEvent event) {
                System.out.println("已经触发了" + event.getType() + "事件！");
            }
        });

    }

    // 创建目录节点
    @Test
    public void createNode() throws Exception {
        zk.create("/zkRoot", "10086".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
    }

    // 创建子节点
    @Test
    public void createChildNode() {
        try {
            zk.create("/zkRoot/zkChild", "1008611".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        } catch (KeeperException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    // 显示节点列表
    @Test
    public void listNodes() throws Exception {
        List<String> children = zk.getChildren("/", true);
        System.out.println(children);
    }

    // 获取节点
    @Test
    public void getNode() throws Exception {
        System.out.println(new String(zk.getData("/zkRoot/zkChild", true, null)));
    }

    // 修改节点信息
    @Test
    public void updateNode() throws Exception {
        // version为-1则忽略版本检查
        zk.setData("/zkRoot/zkChild", "1008612".getBytes(), -1);
    }

    // 删除节点
    @Test
    public void deleteNode() {
        // version为-1则忽略版本检查
        try {
            zk.delete("/zkRoot/zkChild", -1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (KeeperException e) {
            e.printStackTrace();
        }
    }

    // 查看节点状态
    @Test
    public void showStatus() throws Exception {
        Stat rt = zk.exists("/zkRoot/zkChild", true);
        if (rt != null) {
            System.out.println(rt);
        } else {
            System.out.println("节点不存在");
        }
    }

    @After
    public void tearDown() throws Exception {
        zk.close();
    }

}


```



- 客户端向zk服务器注册watcher的同时，会将watcher对象存储在客户端的watchManger中。 

- zk服务器触发watcher事件后，会向客户端发送通知，客户端线程从watchManger中调用watcher执行。 

- watch设置后，一旦触发一次立即会失效，如果需要一直监听，需要再次注册！！！



# zk写数据原理



情况一：客户端直接访问的就是 leader

<img src="https://cdn.jsdelivr.net/gh/StuartYang/oss@master/img/202401130212933.png" alt="image-20240113000425921" style="zoom:30%;" />

情况二：客户端访问的是 follower

<img src="https://cdn.jsdelivr.net/gh/StuartYang/oss@master/img/202401130007102.png" alt="image-20240113000651006" style="zoom:30%;" />



无论情况1，2，当集群中数据写入达到半数，就可以向客户端发出写入响应 ack。



# Apache Curator

- 原生 API 的问题

  1. 会话是异步的，需要自己处理，比如使用 CountDownLatch

  2. Watch 徐呀重复注册，不然不能生效

  3. 开发的复杂性比较高

  4. 不支持多借点删除和创建，需要自己去递归



Curator是一个专门解决分布式锁的框架，解决； 原生 Java API 开发分布式锁的问题。



生产环境中安装多数 zk 合适？

安装奇数台。

10 台服务器：3 台 zk

20 台服务器：5 台 zk



