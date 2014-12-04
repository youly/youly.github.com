---
layout: post
title: 使用spring声明式事务
category: 框架
tags: [spring, transaction]
---

###为什么使用事务
事务有四个特性：

1、原子性：事务作为一个整理执行，包含在其中的数据库操作要么全部执行，要么不执行。

2、一致性：修改数据必须满足约束，包含完整性约束、级联回滚、触发器。[wiki传送](http://en.wikipedia.org/wiki/Consistency_(database_systems))

3、隔离性：确定了事务并发时对其他用户和系统的可见性。一般数据库提供了四个隔离级别：串行、不可重复读、读提交、脏读。

4、持久性：事务提交后，被操作数据被永久保存下载，而不是存在内存中。

在交易系统中，所有数据库操作必须要保证原子性、持久性，因此使用事务。

###什么是声明式事务
声明式事务是spring提供事务管理的一种方式，即通过配置来声明事务管理。spring的声明式事务通过AOP + 代理实现。spring提供的另外一种事务管理是编程式事务。

###与编程式事务相比
声明式事务无需编写额外的事务初始化、提交、回滚代码，无需小心翼翼捕获各种异常进而回滚操作，优点是很明显的。但也有缺点，需懂点配置，代码中有异常要转换成运行时异常抛出让上层知道。

注意，如果要回滚数据库操作，一定要抛出运行时异常且不要有任何try catch此异常的代码。

###事务配置

假设你的数据源配置如下：

    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="${jdbc.driverClassName}" />
        <property name="url" value="${jdbc.url}" />
        <property name="username" value="${jdbc.username}" />
        <property name="password" value="${jdbc.password}" />
    </bean>

则只需添加下面的配置就可以使用spring提供的事务管理了：

    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <tx:advice id="txAdvice" transaction-manager="txManager">
        <tx:attributes>
            <tx:method name="update*" propagation="REQUIRED" read-only="false"/>
            <tx:method name="onMessage" propagation="REQUIRED" read-only="false"/>
            <tx:method name="*" propagation="SUPPORTS" read-only="true"/>
        </tx:attributes>
    </tx:advice>

    <aop:config>
        <aop:pointcut id="txPointcut" expression="
            execution(* com.lastww.service..*.*(..)) or
            execution(* com.lastww.listener..*.*(..))"/>
        <aop:advisor advice-ref="txAdvice" pointcut-ref="txPointcut"/>
    </aop:config>

理解上面的配置需要懂点AOP，[点此了解](http://docs.spring.io/spring-framework/docs/current/spring-framework-reference/html/aop.html)。

上面的配置是说，执行com.lastww.service包及其子包或者com.lastww.listener包及其子包的任何方法时启用事务管理。以update、onMessage开头的方法需要开启事务，其他方法事务以只读的方法执行。

###表达式语法
pointcut有：within、args、target等。其中用的最多的是execution， 执行表达式的模式必须符合：

    execution(modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern(param-pattern) throws-pattern?)

modifiers-pattern：修饰符，如public、protected

ret-type-pattern：方法返回值类型

declaring-type-pattern：<em>这个还不知道是什么</em>

name-pattern：方法名模式

param-pattern：参数模式，(..)代表所有参数,(\*)代表一个参数,(\*,String)代表第一个参数为任何值,第二个为String类型.

throws-pattern：异常列表

###参考
1、[Spring Transaction Management](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/transaction.html)

2、[AspectJ切入点语法详解](http://sishuok.com/forum/posts/list/281.html)

