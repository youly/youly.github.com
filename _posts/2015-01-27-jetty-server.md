---
layout: post
title: Jetty源码阅读 - server
category: 源码
tags: [jetty, java, nio]
---

接着上篇[Jetty源码阅读 - connector](/2015/01/26/jetty-connector/)，本文将分析jetty组件中server组件的源码，基于jetty-9.3.0。

###什么是Server

Jetty由若干个组件组成，Server则把这些组件关联起来，对外提供服务的接口，其在jetty整个架构中的位置如下：

![jetty-high-level-architecture](/assets/images/jetty-high-level-architecture.png)

继承自HandlerWrapper，关联connector，server在这两者之间处于一种协调关系。

###Server类图

![jetty server](/assets/images/jetty_server.jpg)

其中蓝色的线代表“类继承”，绿色实线代表“接口继承”，绿色虚线代表“接口实现”。

从上图可以看到，server和handler的关系与server和connector的关系不同， server类和handler类是继承关系，而不是关联关系。

###Server源码分析

1）server类初始化，需要四个信息，监听ip，监听port，connector组件以及线程池。如下是其中的一个构造函数（文件org.eclipse.jetty.server.Server）：

    public Server(@Name("address")InetSocketAddress addr)
    {
        this((ThreadPool)null);
        ServerConnector connector=new ServerConnector(this);
        connector.setHost(addr.getHostName());
        connector.setPort(addr.getPort());
        setConnectors(new Connector[]{connector});
    }

2）默认的线程池实现是org.eclipse.jetty.util.thread.QueuedThreadPool，线程池中的每个线程不断地从队列中取任务，然后执行。队列采用的是ConcurrentLinkedQueue。

3）Server类继承了ContainerLifeCycle，因此它也有容器的属性，包括启动、停止等。它也管理着与容器生命周期有关的bean，如threadpool，connector。当server类start的时候，会先调用threadpool、connector的start方法。

4）当connector组件接收一个连接并解析完http数据时，会将解析完后封装到HttpChannel的请求信息交给server处理。server则调用handler方法：

文件org.eclipse.jetty.server.Server：

    public void handle(HttpChannel connection) throws IOException, ServletException
    {
        final String target=connection.getRequest().getPathInfo();
        final Request request=connection.getRequest();
        final Response response=connection.getResponse();

        if (LOG.isDebugEnabled())
            LOG.debug(request.getDispatcherType()+" "+request.getMethod()+" "+target+" on "+connection);

        if ("*".equals(target))
        {
            handleOptions(request,response);
            if (!request.isHandled())
                handle(target, request, request, response);
        }
        else
            handle(target, request, request, response);

        if (LOG.isDebugEnabled())
            LOG.debug("RESPONSE "+target+"  "+connection.getResponse().getStatus()+" handled="+request.isHandled());
    }

文件org.eclipse.jetty.server.handler.HandlerWrapper：

    @Override
    public void handle(String target, Request baseRequest, HttpServletRequest request, HttpServletResponse response) throws IOException, ServletException
    {
        if (_handler!=null && isStarted())
        {
            _handler.handle(target,baseRequest, request, response);
        }
    }

###参考

1、[Jetty Architecture](http://www.eclipse.org/jetty/documentation/current/architecture.html#basic-architecture)

