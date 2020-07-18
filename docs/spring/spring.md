# spring 面试题

## spring bean 的生命周期

在一个 bean 实例被初始化时，需要执行一系列的初始化操作已达到可用状态。当一个 bean 不在被调用时需要进行相关的析构操作，并从 bean 容器中移除。

- 实例化阶段
- 属性赋值阶段
- 初始化阶段
- 销毁阶段

## spring bean 的作用域

spring 容器中的 bean 可以分为 5 个范围

- singleton：单例方式，默认，不管多少请求，容器中只有一个 bean 的实例。
- prototype：原型方式，每个请求都会提供一个实例。
- request：在请求 bean 范围内会每一个来自客户端的网格创建一个实例，在请求完成以后，bean会失效并被立即回收。
- session：在 request 类似，确保每个 session 中有一个 bean 的实例，在 session 过期后，bean 会失效并被回收。
- global-session：应用到Portlet应用程序。基于Servlet的应用程序和会话相同。

## 单例 bean 是线程安全吗？

spring 不对单例 bean 做任何多线程的处理。单例的 bean 的并发和线程安全是开发的问题。spring 只是实例化一个对象，并不处理对象内的线程安全。

## spring 事务的实现方式和实现原理

spring 事务的本质其实就是数据库对事务的支持，没有数据库事务支持，spring 是无法提供事务功能的。真正的数据库层的事务提交和回滚是通过 binlog 或者 redo log 实现的。

- 事务的种类
  - 编程式事务管理使用 `TransactionTemplate`
  - 声明式事务使用注解 `@Transactional` ，其本质通过AOP功能，对方法进行拦截。

- 事务的传播行为（多个事务存在的情况下）
  - PROPAGATION_REQUIRED：如果当前没有事务，则创建一个事务，如果存在事务，则加入该事务。（最常用）
  - PROPAGATION_SUPPORTS：如果当前存在事务，则加入该事务，如果不存在，则以非事务执行。
  - PROPAGATION_MANDATORY：如果当前存在事务，则加入该事务，如果不存在，就抛出异常。
  - PROPAGATION_REQUIRES_NEW：创建新事务，无论当前存不存在事务，都创建新事务。
  - PROPAGATION_NOT_SUPPORTED：以非事务方式执行，如果当前存在事务，就把当前事务挂起。
  - PROPAGATION_NEVER：以非事务方式运行，如果当前存在事务，则抛出异常。
  - PROPAGATION_NESTED：如果当前存在事务，则在嵌套事务内执行，如果没有事务，则按PROPAGATION_REQUIRED 类型执行。

- 事务隔离级别
  - ISOLATION_DEFAULT：默认隔离级别，使用数据库默认的事务隔离级别。
  - ISOLATION_READ_UNCOMMITTED：读未提交，允许另外一个事务可以看到这个事务未提交的数据。
  - ISOLATION_READ_COMMITTED：读已提交，保证一个事务修改的数据提交后才能被另一事物读取，而且能看到该事务对已有记录的更新。
  - ISOLATION_REPEATABLE_READ：可重复读，保证一个事务修改的数据提交后才能被另一事务读取，但是不能看到该事务对已有记录的更新。
  - ISOLATION_SERIALIZABLE：一个事务在执行的过程中完全看不到其他事务对数据库所做的更新。

## spring AOP 中的名词解释

- 切面(aspect)：被抽取的公共模块，可能会横切多个对象。
- 连接点(Join Point)：指方法，一个连接点总是代表一个方法的执行。
- 通知(Advice)：在切面的某个连接点上执行的动作。
- 切入点(Pointcut)：致我们要多哪些连接点进行拦截的定义。
- 引入(Introduction)：声明额外的方法或者某个类型的字段。
- 目标对象(Target Object)：被一个或者多个切面所通知的对象。
- 织入(Weaving)：把增加应用到目标对象来创建新的代理对象的过程。

## spring 通知类型

- 前置通知（Before advice）：在连接点之前执行的通知
- 返回后通知（After returning advice）：在连接点正常完成后执行的通知
- 抛出异常通知（After throwing advice）：在方法抛出异常退出时执行的通知
- 后置通知（After advice）在连接点退出时执行的通知（不管正常或者是异常退出）
- 环绕通知（Aroud adcive）：包围一个连接点的通知。在方法调用前后完成的自定义行为，会选择是否继续执行连接点或者直接返回他们自己的返回值或抛出异常来结束执行。

