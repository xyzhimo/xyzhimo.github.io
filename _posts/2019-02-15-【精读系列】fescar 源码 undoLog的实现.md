---
layout: post
title: 【精读系列】fescar 源码 undoLog 的实现
date: 2019-02-15 19:08:13.000000000 +09:00
categories: 精读系列
keyword: fescar
---

### 精读目的

- undoLog 在 fescar 框架中有什么作用？<br>

**答：** fescar 的理念是 90% 的情况下都是提交，所以分支本地事务是在第一阶段就提交了。这时如果其他分支事务出现了异常，需要已经提交的分支事务进行回滚。这个时候，undoLog 就体现它的作用了 —— 依据 undoLog 存储的前后镜像数据，我们可以将数据进行回滚。

### 阅读内容

#### 1. undoLog 的格式

```json
{
    "branchId": 641789253,
    "undoItems": [{
        "afterImage": {
            "rows": [{
                "fields": [{
                    "name": "id",
                    "type": 4,
                    "value": 1
                }, {
                    "name": "name",
                    "type": 12,
                    "value": "GTS"
                }, {
                    "name": "since",
                    "type": 12,
                    "value": "2014"
                }]
            }],
            "tableName": "product"
        },
        "beforeImage": {
            "rows": [{
                "fields": [{
                    "name": "id",
                    "type": 4,
                    "value": 1
                }, {
                    "name": "name",
                    "type": 12,
                    "value": "TXC"
                }, {
                    "name": "since",
                    "type": 12,
                    "value": "2014"
                }]
            }],
            "tableName": "product"
        },
        "sqlType": "UPDATE"
    }],
    "xid": "xid:xxx"
}
```

#### 2. undoLog 存储源码剖析

在抽象类 `AbstractDMLBaseExecutor` 的 `executeAutoCommitFalse()` 方法中我们可以看到 undoLog 构建 beforeImage 和 afterImage 的代码。

```java
protected T executeAutoCommitFalse(Object[] args) throws Throwable {
    // 交由子类 InsertExecutor, UpdateExecutor, DeleteExecutor 去执行这段逻辑
    TableRecords beforeImage = beforeImage();
    T result = statementCallback.execute(statementProxy.getTargetStatement(), args);
    // 同样交由子类 InsertExecutor, UpdateExecutor, DeleteExecutor 去执行这段逻辑
    TableRecords afterImage = afterImage(beforeImage);
    prepareUndoLog(beforeImage, afterImage);
    return result;
}
```

```java
// 这里只以 UpdateExecutor 的代码做文字解释，不做代码解释，具体代码的功能其实很简单，但是代码写起来会很复杂。其他 InsertExecutor, DeleteExecutor 的代码比 UpdateExecutor 更简单。
@Override
protected TableRecords beforeImage() throws SQLException {
    // 省略所有代码，这里只做一下文字解释吧
    // 根据要执行业务 sql 语句, 找到 where 条件, 再将其 select 出来
    ...
    return beforeImage;
}

@Override
protected TableRecords afterImage(TableRecords beforeImage) throws SQLException {
    // 省略所有代码，这里只做一下文字解释吧
    // 根据 beforeImage 的 pkRows, 创建 where primary key 的语句，记录修改后的字段值
    ...
    return afterImage;
}
```

准备 undoLog，根据 beforeImage, afterImage 构造一个 SQLUndoLog 对象。

```java
protected void prepareUndoLog(TableRecords beforeImage, TableRecords afterImage) throws SQLException {
    if (beforeImage.getRows().size() == 0 && afterImage.getRows().size() == 0) {
        return;
    }

    ConnectionProxy connectionProxy = statementProxy.getConnectionProxy();

    TableRecords lockKeyRecords = sqlRecognizer.getSQLType() == SQLType.DELETE ? beforeImage : afterImage;
    String lockKeys = buildLockKey(lockKeyRecords);
    // 构建锁, 最终放入内存的一个 List<String> lockKeysBuffer, 一个 lockKey 格式为 tableName:pks
    connectionProxy.appendLockKey(lockKeys);

    SQLUndoLog sqlUndoLog = buildUndoItem(beforeImage, afterImage);
    // 构建 SQLUndoLog, 最终放入内存的一个 List<SQLUndoLog> sqlUndoItemsBuffer, 一个业务 sql 语句就是一个 SQLUndoLog
    connectionProxy.appendUndoLog(sqlUndoLog);
}
```

准备好了 undoLogs 后，我们就开始需要提交事务了，这里会把 UndoLogs 持久化到数据库中。

```java
@Override
public void commit() throws SQLException {
    if (context.inGlobalTransaction()) {
        try {
            register();
        } catch (TransactionException e) {
            recognizeLockKeyConflictException(e);
        }

        try {
            if (context.hasUndoLog()) {
                // 冲刷 undoLog
                UndoLogManager.flushUndoLogs(this);
            }
            targetConnection.commit();
        } catch (Throwable ex) {
            report(false);
            if (ex instanceof SQLException) {
                throw (SQLException) ex;
            } else {
                throw new SQLException(ex);
            }
        }
        report(true);
        context.reset();
    } else {
        targetConnection.commit();
    }
}
```

创建分支事务锁的字符串，将其传给 TC 端。

```java
private void register() throws TransactionException {
    Long branchId = DataSourceManager.get().branchRegister(BranchType.AT, getDataSourceProxy().getResourceId(),
            null, context.getXid(), context.buildLockKeys());
    context.setBranchId(branchId);
}

public String buildLockKeys() {
    if (lockKeysBuffer.isEmpty()) {
        return null;
    }
    StringBuffer appender = new StringBuffer();
    Iterator<String> iterable = lockKeysBuffer.iterator();
    while (iterable.hasNext()) {
        appender.append(iterable.next());
        if (iterable.hasNext()) {
            appender.append(";");
        }
    }
    return appender.toString();
}
```

冲刷 undoLogs, 将内存的 undoLogs 持久化至数据库中。

```java
public static void flushUndoLogs(ConnectionProxy cp) throws SQLException {
    assertDbSupport(cp.getDbType());

    ConnectionContext connectionContext = cp.getContext();
    String xid = connectionContext.getXid();
    long branchID = connectionContext.getBranchId();

    BranchUndoLog branchUndoLog = new BranchUndoLog();
    branchUndoLog.setXid(xid);
    branchUndoLog.setBranchId(branchID);
    // 把 List<SQLUndoLog> sqlUndoItemsBuffer 放入 branchUndoLog 中
    branchUndoLog.setSqlUndoLogs(connectionContext.getUndoItems());

    String undoLogContent = UndoLogParserFactory.getInstance().encode(branchUndoLog);

    PreparedStatement pst = null;
    try {
        pst = cp.getTargetConnection().prepareStatement(INSERT_UNDO_LOG_SQL);
        pst.setLong(1, branchID);
        pst.setString(2, xid);
        pst.setBlob(3, BlobUtils.string2blob(undoLogContent));
        pst.executeUpdate();
    } catch (Exception e) {
        if (e instanceof SQLException) {
            throw (SQLException) e;
        } else {
            throw new SQLException(e);
        }
    } finally {
        if (pst != null) {
            pst.close();
        }
    }
}
```

#### 3. undoLog 恢复源码剖析

当 RM 收到 TC 发起的分支事务回滚请求时，RM 会取出 undoLog，进行 undo 操作。操作大概如下：
1. 根据 xid, branchId 找到 branchUndoLog。
2. 从 branchUndoLog 循环取出每个 SQLUndoLog 拼接成 undoLog 的sql语句
3. 删除 branchUndoLog。

```java
@Override
public BranchStatus branchRollback(String xid, long branchId, String resourceId, String applicationData) throws TransactionException {
    DataSourceProxy dataSourceProxy = get(resourceId);
    if (dataSourceProxy == null) {
        throw new ShouldNeverHappenException();
    }
    try {
        UndoLogManager.undo(dataSourceProxy, xid, branchId);
    } catch (TransactionException te) {
        if (te.getCode() == TransactionExceptionCode.BranchRollbackFailed_Unretriable) {
            return BranchStatus.PhaseTwo_RollbackFailed_Unretryable;
        } else {
            return BranchStatus.PhaseTwo_RollbackFailed_Retryable;
        }
    }
    return BranchStatus.PhaseTwo_Rollbacked;
}
```

开始进行 undo 操作。`UndoLogManager` 的 `undo()` 方法。

```java
public static void undo(DataSourceProxy dataSourceProxy, String xid, long branchId) throws TransactionException {
    assertDbSupport(dataSourceProxy.getTargetDataSource().getDbType());

    Connection conn = null;
    ResultSet rs = null;
    PreparedStatement selectPST = null;
    try {
        conn = dataSourceProxy.getPlainConnection();

        // The entire undo process should run in a local transaction.
        conn.setAutoCommit(false);

        // Find UNDO LOG
        selectPST = conn.prepareStatement(SELECT_UNDO_LOG_SQL);
        selectPST.setLong(1, branchId);
        selectPST.setString(2, xid);
        rs = selectPST.executeQuery();

        while (rs.next()) {
            Blob b = rs.getBlob("rollback_info");
            String rollbackInfo = StringUtils.blob2string(b);
            BranchUndoLog branchUndoLog = UndoLogParserFactory.getInstance().decode(rollbackInfo);

            for (SQLUndoLog sqlUndoLog : branchUndoLog.getSqlUndoLogs()) {
                TableMeta tableMeta = TableMetaCache.getTableMeta(dataSourceProxy, sqlUndoLog.getTableName());
                sqlUndoLog.setTableMeta(tableMeta);
                AbstractUndoExecutor undoExecutor = UndoExecutorFactory.getUndoExecutor(dataSourceProxy.getDbType(), sqlUndoLog);
                // 见如下代码分析
                undoExecutor.executeOn(conn);
            }
        }
        // 删除数据库的 undoLog
        deleteUndoLog(xid, branchId, conn);
        // 事务提交
        conn.commit();
    } catch (Throwable e) {
        if (conn != null) {
            try {
                conn.rollback();
            } catch (SQLException rollbackEx) {
                LOGGER.warn("Failed to close JDBC resource while undo ... ", rollbackEx);
            }
        }
        throw new TransactionException(BranchRollbackFailed_Retriable, String.format("%s/%s", branchId, xid), e);

    } finally {
        // 释放资源
        ...
    }

}
```

`AbstractUndoExecutor` 的 `executeOn()` 方法。这个方法梳理了大概的执行流程，具体的执行逻辑由子类自己去实现。

```java
public void executeOn(Connection conn) throws SQLException {
    dataValidation(conn);
    try {
        // 根据 sql 的类型来重新构建 undoSql
        String undoSQL = buildUndoSQL();
        PreparedStatement undoPST = conn.prepareStatement(undoSQL);
        // update 和 delete 返回的是 beforeImage, insert 返回的是 afterImage（因为需要把它删除）
        TableRecords undoRows = getUndoRows();
        // 根据每一行的数据不同
        for (Row undoRow : undoRows.getRows()) {
            ArrayList<Field> undoValues = new ArrayList<>();
            Field pkValue = null;
            for (Field field : undoRow.getFields()) {
                if (field.getKeyType() == KeyType.PrimaryKey) {
                    pkValue = field;
                } else {
                    undoValues.add(field);
                }
            }
            // pst set into the value
            undoPrepare(undoPST, undoValues, pkValue);
            // execute
            undoPST.executeUpdate();
        }
    } catch (Exception ex) {
        if (ex instanceof SQLException) {
            throw (SQLException) ex;
        } else {
            throw new SQLException(ex);
        }
    }
}
```

下面的是 `MySQLUndoUpdateExecutor` 的 `buildUndoSQL()` 方法，这个方法主要是要把 sql 恢复到 beforeImage 时的数据。

```java
@Override
protected String buildUndoSQL() {
    // 获取 sql 修改前的镜像数据
    TableRecords beforeImage = sqlUndoLog.getBeforeImage();
    // 获取 sql 修改的行数据
    List<Row> beforeImageRows = beforeImage.getRows();
    if (beforeImageRows == null || beforeImageRows.size() == 0) {
        throw new ShouldNeverHappenException("Invalid UNDO LOG");
    }
    // 随便拿一个行，拼成字段名，比如 update tableName set name = ?, age = ? where pk = ?
    Row row = beforeImageRows.get(0);
    StringBuffer mainSQL = new StringBuffer("UPDATE " + sqlUndoLog.getTableName() + " SET ");
    StringBuffer where = new StringBuffer(" WHERE ");
    boolean first = true;
    for (Field field : row.getFields()) {
        if (field.getKeyType() == KeyType.PrimaryKey) {
            where.append(field.getName() + " = ?");
        } else {
            if (first) {
                first = false;
            } else {
                mainSQL.append(", ");
            }
            mainSQL.append(field.getName() + " = ?");
        }

    }
    return mainSQL.append(where).toString();
}
```

其他两个 `MySQLUndoInsertExecutor` 和 `MySQLUndoDeleteExecutor` 和这个差不多，感兴趣的话你可以去瞧瞧，我这里就不贴代码了，就讲一下他们做的事情。 `MySQLUndoInsertExecutor` 是 delete afterImage 的 sql 语句。`MySQLUndoDeleteExecutor` 是 insert beforeImage 的 sql 语句。

### 问题解答
- 如果 sql 语句修改了主键值，undoLog 的 afterImage 是否会查不出来这条数据呢？<br>
**答：** 目前的实现机制，是不能支持对主键的修改的。修改主键一般是什么样的业务场景呢？我理解应该不是非常有必要吧？比如，业务上需要修改的那个主键改成一个唯一键，另外自增列，做主键，业务层面不需要关注这个自增列。

### 后续补充
1. TC 锁的原理
2. 隔离性
3. RM 和 TC 的 netty 封装
