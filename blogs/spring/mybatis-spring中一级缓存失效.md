### 写在开始
单纯使用mybatis的时候，针对数据库操作，最后是通过一个defaultSqlSession去操作的，而这个操作并没有针对这个sqlsession进行额外的操作，但是spring-mybatis里，每个mapper接口最终会转换成一个代理对象，最终会调用到mapperproxy的invoke方法。

#### mybatis
sqlSession -》defaultSqlSession -》defaultSqlSession.selectList

#### mybatis-spring
sqlSession -》sqlSessionTemplate -》 sqlSessionTemplate.selectList -》 proxy.invoke


### SqlSessionTemplate
由上可知，造成mybatis缓存失效的根本原因在于**SqlSessionTemplate**，那么我们一起来看一下SqlSessionTemplate的具体实现。

**部分源码如下：**

#### 构造函数
```
public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
    notNull(executorType, "Property 'executorType' is required");

    this.sqlSessionFactory = sqlSessionFactory;
    this.executorType = executorType;
    this.exceptionTranslator = exceptionTranslator;
    this.sqlSessionProxy = (SqlSession) newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[] { SqlSession.class },
        new SqlSessionInterceptor());
  }
```
可以看到，构造函数里构造了一个代理对象**sqlSessionProxy**，我们再来看一下标准的selectList的实现，如下所示：
```
/**
   * {@inheritDoc}
   */
  @Override
  public <E> List<E> selectList(String statement) {
    return this.sqlSessionProxy.<E> selectList(statement);
  }
```
可以看见，最终查询就是调用到了**sqlSessionProxy**，而我们知道sqlSessionProxy是一个代理对象，最终会调用到一个实现了InvocationHandler接口的对象，
所以，我们可以推断出，这个对象就是上文中的**SqlSessionInterceptor**，那么我们来看一下SqlSessionInterceptor的源码，如下所示：
```
private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
        Object result = method.invoke(sqlSession, args);
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
          sqlSession = null;
          Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
          if (translated != null) {
            unwrapped = translated;
          }
        }
        throw unwrapped;
      } finally {
        if (sqlSession != null) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }
```
观察以上源码，我们很容易就可以知道，mybatis一级缓存失效的根本原因在于**finally**代码块内部关闭了sqlSession。



