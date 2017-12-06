# 说明

### 这份文档关于将[hyperledger fabric](https://hyperledger-fabric.readthedocs.io/en/release, "超级账本")的持久化部分由[LevelDB](https://github.com/golang/leveldb)改为依赖[Hadoop](https://hadoop.apache.org)的[HBase](https://hbase.apache.org)，对相关技术细节进行解释，试图完整的描述将fabric所使用的KV数据库转换为大数据存储的过程
---
## 目标
修改相关部分，将存储由KV数据库转为大数据存储
## 相关软件版本
### fabric版本：1.0.0
### Hadoop：2.7.4  
### Hbase：1.2.6
---
## 实现
### fabric提供存储功能部分说明

fabric在实际运行中会实例化四个LevelDB——index、ledgerProvider、historyLeveldb、stateLeveldb（其中stateLeveldb可以替换为CouchDB）。

确实比其他的区块链产品要复杂一些，这是由于fabric自身的多channel多ledger设计所造成的。
上述四个四个数据库实际上都是对leveldbhelper（`/fabric/common/ledger/util/leveldbhelper`）进行封装。

相关组件列举如下：    

package       | path | description
------      | ---  | ----
leveldbhelper | /fabric/common/ledger/util/leveldbhelper | 对goleveldb的封装提供一般KV数据库操作
fsblkstorage  | /fabric/common/ledger/blkstorage/fsblkstorage | 对leveldbhelper进行封装提供区块文件级操作
historyLeveldb | /fabric/core/ledger/kvledger/historydb/historyLeveldb | 封装leveldbhelper提供historydb的相关功能
stateLeveldb  | /fabric/core/ledger/kvledger/txmgmt/statedb/stateLeveldb | 封装leveldbhelper提供statedb的相关功能
kvledger      | /fabric/core/ledger/kvledger | 封装state和history提供PeerLedgerProvider的相关功能


首先，改造leveldbhelper包，使底层调用变为hbase，完成相关的单元测试，确保使用hbase作为存储组件没有问题。

之后，修改对应的其他几个package，并进行单元测试，完成整个存储部分的改写。

最后， 运行LTE测试，确保整个fabric正确运行。

----

### 1.leveldbhelper package
**目标** 完成对gohbase的封装，向上提供与之前相同的功能

**修改内容**

**DB** 数据库实体，包含数据库操作的相关属性和方法，**增加hbase操作需要的相关属性**

| 属性名 | 类型     | 说明 |
| :------------- | :------------- | :----
| table | string | hbase中对应的表名 |
| host  | string | hbase运行的主机名 |
| hbase_ac | gohbase.AdminClient | hbase admin client |
| hbase_c | gohbase.Client | hbase client |

**func CreatDB(conf \*Conf) \*DB** 创建数据库实体的方法，**增加hbase实例化相关配置，包括table, host, hbase_ac, hbase_c**

**func (dbInst \*DB) Open()** 打开数据库的方法，**增加hbase对应table的打开或者新建**

**func (dbInst \*DB) Close()** 关闭数据库的方法，**增加hbase的关闭**

**func (dbInst \*DB) Get(key []byte) ([]byte, error)** 根据所给的键返回对应的值，**修改为从hbase获取对应的值**

**func (dbInst \*DB) Put(key []byte, value []byte, sync bool) error** 向数据库内写入对应的键值对，**修改为向hbase中写入对应的键值对**

**func (dbInst \*DB) Delete(key []byte, sync bool) error** 删除数据库中对应的键值对，**修改为删除hbase中对应的键值对**

**func (dbInst \*DB) GetIterator(startKey []byte, endKey []byte) iterator.Iterator** 根据所给的起始截止位置获得迭代器，**修改为由hbase提供的迭代器**

**func (dbInst \*DB) WriteBatch(batch \*leveldb.Batch, sync bool) error** 批写入，**修改为hbase批写入**

**增加内容**

**func CreatTable(client gohbase.AdminClient, table string, cFamilies []string) error** hbase新建table操作

**func (dbInst \*DB) DeleteTable() error** 删除hbase table操作，多用于测试的cleanup

这里需要使用golang编写的hbase客户端或者API，在本次实现中选择[tsuna/gohbase](https://github.com/tsuna/gohbase)作为要使用的hbase client

----

### 2.fsblkstorage package
**目标** 修改原有的FsBlockstoreProvider,封装leveldbhelper,向上提供基于hbase的块保存服务

**新增内容**

**func (p \*FsBlockstoreProvider) DropTable()** 提供cleanup功能，删除对应的ledger表,多用于测试中

----

### 3.historyleveldb package
**目标** 修改原有HistoryDBProvider,封装leveldbhelper,向上提供基于hbase的historyDB相关服务

**新增内容**

**func (provider \*HistoryDBProvider) DropTable()** 提供cleanup功能，删除对应的historydb,多用于测试中

----

### 4.stateleveldb package
**目标** 修改原有VersionedDBProvider,VersionedDB,封装leveldbhelper,向上提供基于hbase的versionDB相关服务

**新增内容**

**func (provider \*VersionedDBProvider) DropTable()**

**func (vdb \*versionedDB) DropTable()** 提供cleanup功能，删除对应的statedb,多用于测试中

----

### 5.kvledger package
**目标** 修改原有Provider,封装blkstorage.BlockStoreProvider,statedb.VersionedDBProvider,historydb.HistoryDBProvider,向上提供基于hbase的存储服务

**新增内容**

**func (provider \*Provider) DropTable()** 提供cleanup功能,删除provider对应的多个db