# SqlSession获取

1. 调用DefaultSqlSessionFactory的openSession方法可以创建出session

   ```java
   private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
       Transaction tx = null;
       try {
         final Environment environment = configuration.getEnvironment();
         final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
         //通过事务工厂来产生一个事务
         tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
         //生成一个执行器(事务包含在执行器里)
         final Executor executor = configuration.newExecutor(tx, execType);
         //然后产生一个DefaultSqlSession
         return new DefaultSqlSession(configuration, executor, autoCommit);
       } catch (Exception e) {
         //如果打开事务出错，则关闭它
         closeTransaction(tx); // may have fetched a connection so lets call close()
         throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
       } finally {
         //最后清空错误上下文
         ErrorContext.instance().reset();
       }
    }
   ```

2. 使用的实现类为DefaultSqlSession，我们可以通过它来执行增删改查等一系列操作

3. 需要注意的是它不直接支持原生sql的提交，想要使用它提供的接口必须提供一个statementId（它是mapper文件中的namespace+对应的insert|delete|update|select标签的id）

4. 常用方式是通过这个sqlSession获取一个mapper接口的代理类，通过它来进行数据库的访问（本质上这个代理类内部也是解析接口名和方法名找到对应的statementId，然后通过sqlSession发起数据库访问）