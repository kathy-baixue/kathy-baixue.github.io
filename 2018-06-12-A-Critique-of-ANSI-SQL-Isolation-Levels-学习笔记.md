## 《A Critique of ANSI SQL Isolation Levels》学习笔记 ##

学习数据库就绕不开事务的隔离级别。其中，最著名和最传统的当属ANSI SQL定义的隔离级别。
# ANSI SQL isolation level #
## 4个事务隔离级别

ANSI SQL 定义了四个隔离级别。
>  Each isolation level is characterized by the phenomena
> that a transaction is **forbidden to experience** (loose or strict interpretations).

Read uncommitted

Read committed

repeatable reads

serializable

![](https://i.imgur.com/wFxOCTN.jpg)

级别从下往上，由低到高。

## 对应3种phenomena 

Dirty reads

No-repeatable reads

phantom reads


## 用到的一些基本概念  

**history**:
> A history models the interleaved execution of a set of transactions as a
> linear ordering of their operations, such as Reads and Writes
> (i.e., inserts, updates, and deletes) of specific data items.

**conflict**:
> Two operations in a history are said to conflict if they are
> performed by distinct transactions on the same data item and
> at least one of them Writes the data item.

**shorthand notation:**
> Histories consisting of reads, writes, commits, and aborts,
> can be written in a shorthand notation: 'w1[x]' means a
> write by transaction 1 (for example) on data item x (which
> is how a data item is “modified’)

w1[x]：transaction 1 write data item x

r2[x]：transaction 2 read data item x

c1：transaction 1 commit

a1：transaction 1 abort

P1：loose interpretation of phenomenon 1

A1：strict interpretation of phenomenon 1

## 串行化的定义 

以下串行化的定义比ANSI SQL中的定义来说，就是一个清晰的串行化的定义，并且能够通过形式语言或者歧义较小的标记来表达。

**dependency graph**:

> A particular history gives rise to a dependency graph
> (sometimes called a serialization graph) defining the temporal
> data flow among transactions.

**graph nodes**:
 
> The committed transactionsin the history are represented as *graph nodes*. 

**edge**:

> If operation op1 of transaction T1 conflicts with action op2 of
> transaction T2 and if op1 precedes op2 in the history, then
> the pair <op1, op2becomes an *edge* in the dependency
> graph.

**equivalent**:

>  Two histories are *equivalent* if they have the same
> transactions and the same dependency graph.

**serializable**

>  A history is *serializable* if it is equivalent to a *serial* *history* — that is, if
> it has the same dependency graph (inter-transaction temporal
> data flow) as some history that executes transactions one at
> a time in sequence.

# ANSI SQL isolation level的不足 #
## 描述上的 ##

- 不直观

通过可能引起anomaly的现象来定义事务隔离级别，在了解到这种思路之前，只是看几种现象和隔离级别的名称常常令人觉得费解。

到底是允许这样的情况？还是一种定义，要达到相应的隔离级别不允许发生这样的现象？例如：read uncommitted级别会出现dirty read这种情况,read committed是避免产生dirty read。是否达到相应的隔离级别还会有对应phemomenon之外的anomoly产生吗？

而且，由于前三种隔离级别通过现象来定义，但是针对第四种“serializable”却没有定义，容易让人误解，serializable就是前面三种现象都不发生，其实避免了上面三种现象的发生，仍然可能不是serializable的。
"However, the ANSI SQL specifications
do not define the SERIALIZABLE isolation level
solely in terms of these phenomena."

- 定义（含义）模糊


即对几种现象的描述，由于使用的自然语言描述，含义模糊，即使对现象的解读采用strict的方式；而且，从解读出的语义的应用来说，其实loose的解读方式才是描述涵盖情况更多，要出现这种phenominon更容易，对隔离级别来说就会更准确。
。由于语言的二义性存在，需要一种形式化的描述才能清晰说明各种现象的定义。"The English statement of P1 is ambiguous."


总体来说，作者说英语的描述具有二义性，语言模糊，其实，中文的二义性和逻辑性差比英语行文更甚。所以，关于事务的隔离级别（以及几种现象）的描述，大致上是wikipedia、ANSI SQL、《A Critique of ANSI SQL Isolation Levels》中缩写符号描述，越往后越准确清楚。

## 跟对应的锁的隔离级别定义的比较 ##

对于隔离级别的描述有多种方式，ANSI的这种就是通过禁止哪些现象来描述，就叫做anomaly-based，其对应的定义叫做phenomenological definitions；通过锁的角度来定义就叫做lock-based。
关于从锁的角度来定义隔离级别，讲了好几个基本概念。

- ANSI的跟用锁来定义隔离级别比较起来较弱

文章提到了另一篇论文[OOBBGM]，里面有证明。下面举了个例子说明，即使在最低的隔离级别READ UNCOMMITTED中提到的Dirty write现象，由于锁隔离级别的long during write特性，也可以避免这种可能导致anomaly的现象，而ANSI的定义里面没有排除到这种现象。

- ANSI SQL对于现象的解读，Loose interpretation才是正确的解读。

  以Dirty read为例子：
  strict interpretation 记为A1：w1[x]...r2[x]...(a1 and c2 in either order)

  loose interpretation 记为P1 ：w1[x]...r2[x]...((c1 or a1) and (c2 or a2) in any order) 

  historty:H1: r1[x=50]w1[x=10]r2[x=10]r2[y=50]c2r1[y=50]w1[y=90]c1

 T2 reads an inconsistent state where the total balance is 60.事务2（T2）读到了一个不一致的结果 x + y = 60,违背了数据的一致性。
 The history H1 does not violate any of the anomalies A1、A2、A3；
 H1 indeed violates P1.
 所以P1的loose interpretation可以，所以ANSI的phenominon的解读应该是loose interpretation想要的意思。
# isolation level #
关于隔离级别的强弱的定义和符号表示

L1弱于L2（或L2强于L1）
> L1 is weaker than isolation level L2 (or L2 is stronger than L1), denoted L1 « L2, if all
> non-serializable histories that obey the criteria of L2 also
> satisfy L1 and there is at least one non-serializable history
> that can occur at level L1 but not at level L2. L1 is no stronger than L2, denoted L1 « L2 if either
> L1 « L2 or L1 == L2.

L1等于L2

> Two isolation levels L1 and L2 are equivalent, denoted L1 == L2, when the
> sets of non-serializable histories satisfying L1 and L2 are
> identical. 

两个隔离级别不可比

>  Two isolation levels are incomparable,
> denoted L1 »« L2, when each isolation level allows
> a non-serializable history that is disallowed by the other.

补充说明：上面关于两个隔离级别的差别都是按可能会发生的不串行history来定义，其实两个级别的差异还体现在不允许的串行化的历史，locking serializable尽管也存在有不允许的serializable histories，但是我们认为Locking serializable是串行化的。

关于隔离级别的对比定义比较精确了，同时也说明了尽管有的serializable history不被lock scheduler（lock隔离级别的实现依赖于scheduler）允许，但是locking serializable也认为是串行化的，这就相当于是形成了一种约定。

“Transactions executing under a *locking scheduler* request
Read (Share) and Write (Exclusive) locks on data items or
sets of data items they read and write.”

# 其他几个事务的隔离级别 #

关于以下两个隔离级别跟ANSI SQL级别的强度对比和引起或解决新的anomaly，文章中有详细说明。
## cursor stability ##
cursor stability的设计是为了防止lost update phenomenon.
隔离级别强度介于READ COMMITTED 和 REPEATABLE READ之间。很多应用通过cursor stability 避免锁的竞争。

## snapshot isolation ##

snapshot isolation（缩写为si）是一种MVCC（multiversion concurrency control）。si能够使事务得到一个在事务开始时间Start-Timestamp的数据库的consistent snapshot；并且该事务能够成功提交的条件从开始时间得到的snapshot起没有并发的修改与本事务修改发生冲突。

- 事务处于si的特点

1、不会阻塞读请求

2、本事务自己做出的修改会反映在事务开始时的snapshot上，即如果需要再次读数据，需要重新获取数据。

3、其他事务在本事务开始时间后做出的修改对本事务不可见



相关资源链接：

[A Critique of ANSI SQL Isolation Levels](https://www.cs.umb.edu/cs734/CritiqueANSI_Iso.pdf)

[A History of Transaction Histories](https://ristret.com/s/f643zk/history_transaction_histories)

鸣谢：资源来源
[何_登成](https://www.weibo.com/u/2216172320?from=feed&loc=nickname)
微博
[https://www.weibo.com/2216172320/Gaqjeiulo](https://www.weibo.com/2216172320/Gaqjeiulo "何登成技术文章推荐")