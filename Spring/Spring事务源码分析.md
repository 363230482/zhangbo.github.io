org.springframework.transaction.interceptor.BeanFactoryTransactionAttributeSourceAdvisor

#Spring事务处理流程
```
spring事务传播机制
REQUIRED：支持当前事务，如当前没有事务，则新开一个事务
SUPPORTS：支持当前事务，如当前没有事务，则以无事务方式执行
MANDATORY：支持当前事务，如当前没有事务，则抛一个异常
REQUIRES_NEW：如当前存在事务，则挂起当前事务，在新开一个事务；如当前不没有事务，则同样新开一个事务
NOT_SUPPORTED：如当前存在事务，则挂起当前事务，以无事务的方式执行
NEVER：不支持事务，如当前存在事务，则抛异常
NESTED：嵌套事务，以savepoint(JDBC事务)的方式在当前事务中执行(JTA事务则新开一个事务)
```

## 1.invokeWithinTransaction() 部分源码如下：
```
org.springframework.transaction.interceptor.TransactionAspectSupport#invokeWithinTransaction

// 创建一个事务
TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
Object retVal = null;
try {
   // This is an around advice: Invoke the next interceptor in the chain.
   // This will normally result in a target object being invoked.
// 执行下一个aop，或者是原本的业务逻辑，并获取业务返回值
   retVal = invocation.proceedWithInvocation();
}
catch (Throwable ex) {
    // 事务回滚，默认回滚RuntimeExcetpion和Error，指定rollbackFor除外
   completeTransactionAfterThrowing(txInfo, ex);
   throw ex;
}
finally {
   cleanupTransactionInfo(txInfo);
}
// 提交事务
commitTransactionAfterReturning(txInfo);
return retVal;
```

## 2.开启事务
```
org.springframework.transaction.interceptor.TransactionAspectSupport#createTransactionIfNecessary

org.springframework.transaction.support.AbstractPlatformTransactionManager#getTransaction

org.springframework.jdbc.datasource.DataSourceTransactionManager#doGetTransaction (new 一个 DataSourceTransactionObject)
org.springframework.jdbc.datasource.DataSourceTransactionManager#isExistingTransaction (判断DataSourceTransactionObject是否存在ConnectionHolder且有活跃事务)
```

### 2.1无事务时新开事务
```
// 1.无事务
org.springframework.transaction.support.AbstractPlatformTransactionManager#startTransaction
org.springframework.jdbc.datasource.DataSourceTransactionManager#doBegin

// ===== doBegin ===== //
// 开始一个新事务
protected void doBegin(Object transaction, TransactionDefinition definition) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    Connection con = null;

    try {
        if (!txObject.hasConnectionHolder() ||
                txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
            Connection newCon = obtainDataSource().getConnection();
            if (logger.isDebugEnabled()) {
                logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
            }
            // 绑定当前的connection到当前线程
            txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
        }
        // 设置事务开始标志
        txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
        con = txObject.getConnectionHolder().getConnection();
        // 设置事务隔离级别
        Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
        txObject.setPreviousIsolationLevel(previousIsolationLevel);
        txObject.setReadOnly(definition.isReadOnly());

        // Switch to manual commit if necessary. This is very expensive in some JDBC drivers,
        // so we don't want to do it unnecessarily (for example if we've explicitly
        // configured the connection pool to set it already).
        if (con.getAutoCommit()) {
            txObject.setMustRestoreAutoCommit(true);
            if (logger.isDebugEnabled()) {
                logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
            }
            // *****设置手动提交事务*****
            con.setAutoCommit(false);
        }

        prepareTransactionalConnection(con, definition);
        // 设置事务为活跃状态，在判断是否存在一个事务时使用（处理事务传播机制时）
        txObject.getConnectionHolder().setTransactionActive(true);

        int timeout = determineTimeout(definition);
        if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
            txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
        }

        // Bind the connection holder to the thread.
        if (txObject.isNewConnectionHolder()) {
            TransactionSynchronizationManager.bindResource(obtainDataSource(), txObject.getConnectionHolder());
        }
    }

    catch (Throwable ex) {
        if (txObject.isNewConnectionHolder()) {
            DataSourceUtils.releaseConnection(con, obtainDataSource());
            txObject.setConnectionHolder(null, false);
        }
        throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
    }
}
// ===== doBegin end ===== //
```
### 2.2已存在事务，根据传播机制判断
```
2.已存在事务
org.springframework.transaction.support.AbstractPlatformTransactionManager#handleExistingTransaction
// ===== handleExistingTransaction ===== //
通过比较TransactionDefinition.PROPAGATION_REQUIRES_NEW，确定是新开事务或者其他方式

private TransactionStatus handleExistingTransaction(
        TransactionDefinition definition, Object transaction, boolean debugEnabled)
        throws TransactionException {
    // 不支持事务
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
        throw new IllegalTransactionStateException(
                "Existing transaction found for transaction marked with propagation 'never'");
    }

    // 当前不支持事务，挂起上一个事务
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
        if (debugEnabled) {
            logger.debug("Suspending current transaction");
        }
        Object suspendedResources = suspend(transaction);
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(
                definition, null, false, newSynchronization, debugEnabled, suspendedResources);
    }

    // 新开一个事务
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
        if (debugEnabled) {
            logger.debug("Suspending current transaction, creating new transaction with name [" +
                    definition.getName() + "]");
        }
        SuspendedResourcesHolder suspendedResources = suspend(transaction);
        try {
            return startTransaction(definition, transaction, debugEnabled, suspendedResources);
        }
        catch (RuntimeException | Error beginEx) {
            resumeAfterBeginException(transaction, suspendedResources, beginEx);
            throw beginEx;
        }
    }

    // 嵌套事务，使用savepoint保存一个保存点用于回滚
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        if (!isNestedTransactionAllowed()) {
            throw new NestedTransactionNotSupportedException(
                    "Transaction manager does not allow nested transactions by default - " +
                    "specify 'nestedTransactionAllowed' property with value 'true'");
        }
        if (debugEnabled) {
            logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
        }
        if (useSavepointForNestedTransaction()) {
            // Create savepoint within existing Spring-managed transaction,
            // through the SavepointManager API implemented by TransactionStatus.
            // Usually uses JDBC 3.0 savepoints. Never activates Spring synchronization.
            DefaultTransactionStatus status =
                    prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
            status.createAndHoldSavepoint();
            return status;
        }
        else {
            // Nested transaction through nested begin and commit/rollback calls.
            // Usually only for JTA: Spring synchronization might get activated here
            // in case of a pre-existing JTA transaction.
            return startTransaction(definition, transaction, debugEnabled, null);
        }
    }

    // Assumably PROPAGATION_SUPPORTS or PROPAGATION_REQUIRED.
    if (debugEnabled) {
        logger.debug("Participating in existing transaction");
    }
    if (isValidateExistingTransaction()) {
        if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
            Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
            if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
                Constants isoConstants = DefaultTransactionDefinition.constants;
                throw new IllegalTransactionStateException("Participating transaction with definition [" +
                        definition + "] specifies isolation level which is incompatible with existing transaction: " +
                        (currentIsolationLevel != null ?
                                isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :
                                "(unknown)"));
            }
        }
        if (!definition.isReadOnly()) {
            if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
                throw new IllegalTransactionStateException("Participating transaction with definition [" +
                        definition + "] is not marked as read-only but existing transaction is");
            }
        }
    }
    boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
    return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
}

```

## 3.回滚事务
```
// 处理回滚事务
org.springframework.transaction.interceptor.TransactionAspectSupport#completeTransactionAfterThrowing
// 判断是否需要回滚，如不需要回滚，则提交事务
org.springframework.transaction.interceptor.RuleBasedTransactionAttribute#rollbackOn
org.springframework.transaction.interceptor.DefaultTransactionAttribute#rollbackOn
// 通过判断当前跑出的异常是否是指定的回滚异常的子类
org.springframework.transaction.interceptor.RollbackRuleAttribute#getDepth(java.lang.Throwable)
```
