# Tidb源码阅读-Delete语句的执行浅析 #

本文是参考Tidb源码阅读系列，编写的关于delete语句执行的一点粗浅认识。文章模仿该系列的两篇文章，insert语句和select语句执行，尝试在此基础上进行delete语句执行的分析。

文章要感谢申砾的指导，感谢彦青。文章水平有限，不足的地方请见谅，感谢Tidb创造了平台和机遇，让热爱数据库的伙伴们，参加社区活动和讨论，一起提高对分布式数据库实现的认识。


----------
## Tidb源码阅读简述  ##
Tidb源码的特点，大部分包以接口的形式对外提供服务，并且大部分功能也集中在某个包中。例如：包executor，接口Executor的定义在该包中，该包中的大部分功能都实现了接口Executor的方法，例如：Next（），NextChunk（）等。再如：包tidb中的session.go文件中大部分都实现Session接口的方法，以接口形式对外提供服务。

   源码分析方法从两个方面入手，方法在什么地方被调用，具体的定义在哪个包。理解调用关系的时候可以借鉴stack diagram方法，优点是可以清楚看出的参数的赋值情况和方法的调用关系。包的分布可以从tidb的3个层次的每个层次的功能入手，根据功能职责划分形成包，再看包当中的接口的情况，本文主要分析的是SQL层。
## 1. Protocol layer协议转化(command Query to SQL Text) ##
- 主要功能：管理客户端连接；解析MySQL命令并返回执行结果。
- 协议层入口：clientConn.Run
- 需要关注的方法：dispatch、handleQuery、cc.ctx.Execute、session.Execute。
- 协议层出口：如果是像select语句需要将结果写回客户端的，调用writeResultSet。
- 其他说明参考注释：All the related codes are in the server package.For a single connection, the entry point method is the dispatch of the clientConn class, where the protocol is parsed and different processing functions are called.
## 2. SQL layer ##

- 主要功能：SQL语句的语法解析、合法性验证、制定查询计划、优化查询计划、根据计划生成查询器、执行并返回结果。

- 其他说明参考注释：The entry point of the process is in the session.go file. TiDB-Server calls the Session.Execute() interface, inputs the SQL statement and implements Session.Execute().

  执行经过验证和优化后的物理计划，Session.executeStatement调用runStmt，执行语句，返回结果集。语句如下：recordSet, err := runStmt(ctx, s, stmt)。runStmt定义见session/tidb.go。runStmt通过调用接口ast.Statement的Exec方法执行，语句如下：rs, err = s.Exec(ctx)。Exec方法的定义见executor/adapter.go，ExecStmt结构实现ast.Statement接口，所以Exec方法定义语句如下：func (a *ExecStmt) Exec(ctx context.Context) (ast.RecordSet, error){…}。

 由于executor不向客户端返回结果集，调用handleNoDelayExecutor方法，其定义见executor/adapter.go，调用executor接口的Next方法，语句如下：err = e.Next(ctx, e.newChunk())。由于delete语句的executor的类型是DeleteExec，next的定义见executor/write.go，语句如下：func (e *DeleteExec) Next(ctx context.Context, chk *chunk.Chunk) error {…}。
### 2.1解析并构造结构化数据AST（SQL Text to AST） ###
将文本解析成结构化的数据，即抽象语法树（AST）。针对delete语句来说就是将SQL Text转换为DeleteStmt结构。
参考前面关于select、insert语句的处理，简要描述如下：

- Lexer：将文本转换成token，交给Parser。
- Parser：根据token序列匹配语法规则，输出结构化的节点。
- 节点转换：由DeleteStmt. Accept，其实现ast.Node接口的Accept方法，以Visitor模式遍历节点对AST做结构转换。源码见ast/dml.go。
  
注意几个Node接口的关系Node、StmtNode、DmlNode及与DeletStmt关系。涉及的源码见ast/ast.go。

session.go中的一些主要的方法简介。

名称Name | 作用Responsible for |简介Description|定义的源码位置Defined in|相关的定义 Reference defined in 
- | :-: | -:|-:|-:|
session.Execute|按步骤执行SQL语句|Method on session;implement interface Session|session.go|Struct session in session.go.Interface Session in session.go|
session.ParseSQL；compiler.Compile；session.executeStatement|解析SQL语句,生成ast.StmtNode；对AST进行验证、优化,将ast.StmtNode编译成physical plan；执行physical plan|Method on session;Method on Compiler;|session.go;executor/compiler.go;session.go|/; Struct Compiler in executor/compiler.go;/ |

###2.2 制定查询计划以及优化（AST to plan） ###
将ast.DeleteStmt结构转换为plan.Delete结构。

具体在planBuilder. build ()中完成，针对delete由planBuilder.buildDelete（）完成。

涉及的源码：plan/planbuilder.go。planBuilder.buildDelete方法定义见plan/logical_plan_builder.go。

###2.3生成执行器（plan to Executor ）###
将plan.Delete结构转换为executor.DeleteExec结构。由executorBuilder.buildDelete实现，代码见executor/builder.go，定义语句如下：func (b *executorBuilder) buildDelete(v *plan.Delete) Executor {…}。在生成执行器DeleteExec的过程中，生成查询执行器SeleteExec，语句如下：selExec := b.build(v.SelectPlan)，执行器SelectExec封装在DeleteExec中：
 
    deleteExec := &DeleteExec{
    		baseExecutor: newBaseExecutor(b.ctx, nil, v.ExplainID(), selExec),
    		SelectExec:   selExec,
    		Tables:   v.Tables,
    		IsMultiTable: v.IsMultiTable,
    		tblID2Table:  tblID2table,
    	}

### 2.4执行Executor ###
执行Executor，回忆第一篇简介中，executor包作用：执行器的生成及执行。可知这部分代码主要在package executor中。
执行入口是Next，后续主要的执行都是以结构DeleteExec的方法来进行。

“简介”是指对名称里列出的方法或函数的说明；

“定义的源码位置”是指方法或函数在哪里定义，如果对于接口的方法来说，就是指接口中方法的实现。

表格中列出的主要的方法，没有考虑一些判断或特例，都是按照最简单的只删除一张表中的数据来举例。其中，行之间的顺序体现了调用关系，上一行为了完成工作职责调用了下一行的函数，方便以完整的思路关注重点功能。


名称Name | 作用Responsible for |简介Description|定义的源码位置Defined in|相关的定义 Reference defined in 
- | :-: | -:|-:|-:| 
DeleteExec.Next | 执行入口| Method on DeleteExec;implement interface executor.Executor |executor/write.go|Struct DeleteExec in executor/write.go;Interface Executor in executor/executor.go|
DeleteExec.deleteSingleTable | 删除单个表中的数据| Method on DeleteExec|executor/write.go| /|
DeleteExec.deleteOneRow | 删除一行数据| Method on DeleteExec |executor/write.go|/|
DeleteExec.removeRow||Method on DeleteExec|executor/write.go|/|
Table.RemoveRecord|删除数据相关的具体删除操作|Method on Table;implement interface table.Table|table/tables/tables.go|Struct Table in table/tables/tables.go;Interface Table in table/table.go |
Table.removeRowData  Table.removeRowIndices|removeRowData删除行的数据；removeRowIndices 删除一行数据所有的索引|Method on Table|table/tables/tables.go|/|

DeleteExec实现executor接口Next方法，通过deleteMultiTablesByChunk和deleteSingleTableByChunk实现多个表数据和单个表数据的删除，以单表为例。deleteSingleTableByChunk的执行：
 先获取数据tbl, handleCol, datumRow；再调用DeleteExec.deleteOneRow，获取handle := row[handleCol.Index].GetInt64()，然后调用table.Table.removeRecord (ctx, h, data)，再分别调用removeRowData和removeRowIndices删除行的数据和索引。

 Table.removeRowData删除行的数据，先构造row的key，然后调用err := ctx.Txn().Delete([]byte(t.RecordKey(h)))。

Table.removeRowIndices通过调用v.Delete(ctx.GetSessionVars().StmtCtx, ctx.Txn(), vals, h)删除索引。具体通过调用key, _, err := c.GenIndexKey(sc, indexedValues, h, nil) 构造Key，然后调用m.Delete()通过kv接口删除索引err = m.Delete(key)。代码见table/tables/index.go

# 3.	KV API layer #
调用KV接口进行删除。KV接口层负责将请求路由到正确的KV Server，接收返回消息给SQL层。
