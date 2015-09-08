# 表格的创建流程
从teracli开始入手分析。

1. 解析命令行参数，如果是create操作，则交由CreateOp函数执行
2. CreateOp查检参数的合法性，调用ParseSchema对参数进行解析，生成表的schema, 如果命令行参数中有delimiter，则也要对delimiter作解析，然后将解析结果传入ClientImpl::CreateTable函数处理
3. ClientImpl::CreateTable()函数先获得Master的stub，根据上一步解析出来的表的名字，schema, 预设tablet等信息构造CreateTableRequest, 然后调用SendMessageWithRetry进行RPC调用master的CreateTable接口
4. Master收到CreateTable请求后，先检查自身状态，如果不是在kIsRunning状态，则拒绝请求。首先调用TableManager的FindTable接口，来确定需要创建的table是否已经存在，如存在则设置错误信息，并返回，如果开启了acl，则需要检查是否有权限创建表格(好像当前只支持root用户)。检查完毕后，先调用MoveEnvDirToTrash，为创建新的存放表格数据的目录提供环境。然后检查tablet数目是否过多，tablet是否设置错误(要保证tablet之间有序), 以及locality\_groups是否过少(至少是1)。然后依次为每一个tablet构造存放数据的路径、start\_key、end\_key，并调用TabletManager的AddTablet接口，将这些tablet添加到内存中维护的tablet信息表中去
5. 为了持久化表的信息，构建WriteClosure闭包，将table信息(locality\_groups, table\_name, compress\_type等)置于闭包内，调用BatchWriteMetaTableAsync，异步调用RPC，将表及下属的tablet元信息写到元数据表中
6. BatchWriteMetaTableAsync函数被重载成了两种形态，其实一种是对另一种的封装。这里被调用的是未封装的版本。为了方便encode元数据表中的key, value，对于table信息，以及每个tablet，都bind一个encode函数，统一放入ToMetaFunc vector中，然后便调用BatchWriteMetaTableAsync的另外一个版本
7. BatchWriteMetaTableAsync封装版首先获得加载了元数据表的tablet server的stub，然后开始构造WriteTabletRequest(因为元数据也是存在leveldb里的，所以可以像写其它数据一样，直接RPC调用WriteTablet接口), 又由于是元数据，所以is\_sync及is\_instant都设置为了true, 构造完毕后，RPC调用WriteTablet写入元信息

对于上面提到的，对元数据表中key, value进行encode的函数， Table信息使用的是Table::ToMetaTableKeyValue，encode后，key为"@" + table\_name, value为该tablet的名字，schema, 创建时间，快照列表(序列化), Tablet信息使用的是Tablet::ToMetaTableKeyValue，encode后，key为table\_name + "#" + key\_start, value为该tablet的key range, table\_size, compress, store\_medimu等信息(序列化)

对于上面提到的，WiteTablet的具体过程，由于与写其它数据并没有什么不同，等到分析写入流程时，再具体分析