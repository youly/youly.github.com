---
layout: post
title: Jetty源码阅读 - connector
category: 源码
tags: [jetty, java, nio]
---

### Jetty是什么

Jetty是一个开源的项目，它主要提供了一个轻量级的Web Server和Servlet容器。

Jetty中有个重要的数据结构Handler，所有可以被扩展的组件都可以作为一个handler添加到server中，由jetty来管理这些handler。本文将分析jetty组件中connector组件的源码，基于jetty-9.3.0。

### 什么是connnector

Jetty由若干个组件组成，connector是其中负责处理客户端连接的组件，其在jetty整个架构中的位置如下：

![jetty-high-level-architecture](/assets/images/jetty-high-level-architecture.png)

### connector类图

jetty包中主要的类 org.eclipse.jetty.server.Server 类中关联了 ServerConnector 类，ServerConnector类图如下：

![ServerConnector](/assets/images/ServerConnector.jpg)

其中蓝色的线代表“类继承”，绿色实线代表“接口继承”，绿色虚线代表“接口实现”。

### connector源码分析

1、成员变量ServerConnector类在Server类构造时被初始化，如下（文件org.eclipse.jetty.server.Server）：

    public Server(@Name("port")int port)
    {
        this((ThreadPool)null);

        /** 在这里初始化了connector */
        ServerConnector connector=new ServerConnector(this);
        connector.setPort(port);
        setConnectors(new Connector[]{connector});
    } 

2、Server启动的时候会先调用各个组件bean的启动方法，然后是connector的start()方法，文件（org.eclipse.jetty.server.Server ）：

    @Override
    protected void doStart() throws Exception
    {
        //If the Server should be stopped when the jvm exits, register
        //with the shutdown handler thread.
        if (getStopAtShutdown())
            ShutdownThread.register(this);

        //Register the Server with the handler thread for receiving
        //remote stop commands
        ShutdownMonitor.register(this);
        
        //Start a thread waiting to receive "stop" commands.
        ShutdownMonitor.getInstance().start(); // initialize

        //中间代码省去

        try
        {
            super.doStart(); //由父类调用各个组件bean的启动方法
        }
        catch(Throwable e)
        {
            mex.add(e);
        }
        // start connectors last
        for (Connector connector : _connectors)
        {
            try
            {   
                connector.start(); //注意这里
            }
            catch(Throwable e)
            {
                mex.add(e);
            }
        }
        
        if (isDumpAfterStart())
            dumpStdErr();

        mex.ifExceptionThrow();

        LOG.info(String.format("Started @%dms",Uptime.getUptime()));
    }

3、ServerConnector的start方法干了些什么呢？

1）打开并绑定一个ServerSocketChannel（文件org.eclipse.jetty.server.ServerConnector）：

    @Override
    public void open() throws IOException
    {
        if (_acceptChannel == null)
        {
            //中间代码省略
            if (serverChannel == null)
            {
                serverChannel = ServerSocketChannel.open();

                InetSocketAddress bindAddress = getHost() == null ? new InetSocketAddress(getPort()) : new InetSocketAddress(getHost(), getPort());
                serverChannel.socket().setReuseAddress(getReuseAddress());
                serverChannel.socket().bind(bindAddress, getAcceptQueueSize());

                _localPort = serverChannel.socket().getLocalPort();
                if (_localPort <= 0)
                    throw new IOException("Server channel not bound");

                addBean(serverChannel);
            }

            /**
             *  注意此处是阻塞模式
             *  调用serverChannel.accept时将阻塞直到从操作系统获取到连接
             */
            serverChannel.configureBlocking(true);
            addBean(serverChannel);

            _acceptChannel = serverChannel;
        }
    }

2）多个线程（Acceptor）在同一个ServerSocketChannel上accept连接（文件org.eclipse.jetty.server.AbstractConnector）：

        for (int i = 0; i < _acceptors.length; i++)
        {
            Acceptor a = new Acceptor(i);
            addBean(a);
            getExecutor().execute(a);
        }

Acceptor类run方法最终调用到serverChannel.accept()方法，从操作系统获取到客户端的一个连接，之后将此连接交给Selector管理（文件org.eclipse.jetty.server.AbstractConnector）：

            try
            {
                while (isAccepting()) //此处进入一个循环，不停地从操作系统获取已经建立的连接
                {
                    try
                    {
                        accept(_id); //调用serverChannel.accept()获取就绪的连接，阻塞
                    }
                    catch (Throwable e)
                    {
                        if (isAccepting())
                            LOG.warn(e);
                        else
                            LOG.ignore(e);
                    }
                }
            }

accept方法代码如下（文件org.eclipse.jetty.server.ServerConnector）：

    @Override
    public void accept(int acceptorID) throws IOException
    {
        ServerSocketChannel serverChannel = _acceptChannel;
        if (serverChannel != null && serverChannel.isOpen())
        {
            SocketChannel channel = serverChannel.accept();
            accepted(channel);
        }
    }
    
    private void accepted(SocketChannel channel) throws IOException
    {
        channel.configureBlocking(false);
        Socket socket = channel.socket();
        configure(socket);
        _manager.accept(channel);
    }

3）连接交由Selector管理。上面的成员变量_manager在ServerConnector构造时被初始化。SelectorManager管理多个Selector（org.eclipse.jetty.io.                                                                                                                                               ManagedSelector），每次从channel里accept到一个连接时，_manager从selector数组中选取一个ManagedSelector，由此ManagedSelector负责此连接后续数据的读写。

文件org.eclipse.jetty.io.SelectorManager，选择selector，代码接上面的_manager.accept(channel)：

    public void accept(SocketChannel channel)
    {
        accept(channel, null);
    }

    public void accept(SocketChannel channel, Object attachment)
    {
        final ManagedSelector selector = chooseSelector(channel);
        selector.submit(selector.new Accept(channel, attachment)); //这里异步给selector提交一个channel注册请求，将连接的数据读写交给selector统一监听
    }

查看Accept类的run方法，核心代码如下（文件org.eclipse.jetty.io.ManagedSelector）：

    try
    {
        SelectionKey key = channel.register(_selector, 0, attachment);
        EndPoint endpoint = createEndPoint(channel, key);
        key.attach(endpoint);
    }
    catch (Throwable x)
    {
        closeNoExceptions(channel);
        LOG.debug(x);
    }

可以看到这里channel向_selector注册了感兴趣的事件0，为什么是0呢？这个还不知道。除了事件，channel还附加了一个attachment，这个attachment很关键。当select系统调用返回时，数据处理的工作就由attachment来处理了，即http请求的解析，处理，响应。看看createEndPoint干了些什么（文件org.eclipse.jetty.io.ManagedSelector）：

    private EndPoint createEndPoint(SocketChannel channel, SelectionKey selectionKey) throws IOException
    {
        EndPoint endPoint = _selectorManager.newEndPoint(channel, this, selectionKey);
        //新建endpoint时，新建并调度了一个线程，监控连接的空闲时间
        _selectorManager.endPointOpened(endPoint);
        Connection connection = _selectorManager.newConnection(channel, endPoint, selectionKey.attachment());
        endPoint.setConnection(connection);
        //新建connection时，注册了数据可读时的回调函数
        _selectorManager.connectionOpened(connection);
        if (LOG.isDebugEnabled())
            LOG.debug("Created {}", endPoint);
        return endPoint;
    }

当connection的数据可读时，代码将走到下面，文件org.eclipse.jetty.server.HttpConnection：

    @Override
    public void onOpen()
    {
        super.onOpen();
        fillInterested();
    }

文件org.eclipse.jetty.io.AbstractConnection：

    public void fillInterested()
    {
        if (LOG.isDebugEnabled())
            LOG.debug("fillInterested {}",this);
        getEndPoint().fillInterested(_readCallback);
    }

HttpConnection最终调用HttpChannel的handle方法，处理此次连接:

文件org.eclipse.jetty.server.HttpConnection：

    @Override
    public void onFillable()
    {
        //后面的代码省略
        try
        {
                    //后面的代码省略
            while (true)
            {
                    //后面的代码省略
                    boolean suspended = !_channel.handle();
                    //后面的代码省略
                    
            }
                
        }
    }

文件org.eclipse.jetty.server.HttpChannel：

    public boolean handle()
    {
        if (LOG.isDebugEnabled())
            LOG.debug("{} handle {} ", this,_request.getHttpURI());

        HttpChannelState.Action action = _state.handling();
        try
        {
            // Loop here to handle async request redispatches.
            // The loop is controlled by the call to async.unhandle in the
            // finally block below.  Unhandle will return false only if an async dispatch has
            // already happened when unhandle is called.
            loop: while (action.ordinal()<HttpChannelState.Action.WAIT.ordinal() && getServer().isRunning())
            {
                boolean error=false;
                try
                {
                    if (LOG.isDebugEnabled())
                        LOG.debug("{} action {}",this,action);

                    switch(action)
                    {
                        case REQUEST_DISPATCH:
                            if (!_request.hasMetaData())
                                throw new IllegalStateException();
                            _request.setHandled(false);
                            _response.getHttpOutput().reopen();
                            _request.setDispatcherType(DispatcherType.REQUEST);

                            List<HttpConfiguration.Customizer> customizers = _configuration.getCustomizers();
                            if (!customizers.isEmpty())
                            {
                                for (HttpConfiguration.Customizer customizer : customizers)
                                    customizer.customize(getConnector(), _configuration, _request);
                            }
                            getServer().handle(this); //交给server处理
                            break;

    //后面的代码省略

4）ServerConnector的doStart方法执行时，_manager（org.eclipse.jetty.io.SelectorManager）的doStart方法也将被执行：如下（文件org.eclipse.jetty.io.ManagedSelector）：

    @Override
    protected void doStart() throws Exception
    {
        super.doStart();
        for (int i = 0; i < _selectors.length; i++)
        {
            ManagedSelector selector = newSelector(i);
            _selectors[i] = selector;
            selector.start();
            execute(selector);
        }
    }

_manager主要是调度selector任务。selector内部也有一个调度队列，负责异步处理数据读写请求。

5）ManagedSelector类。ManagedSelector的run方法，其主要工作是阻塞等待监听的channel数据读写ready，以及接受新的channel数据监听请求。

如果selector已经处于select状态下，需要先调用selector的wakeup方法，然后才能往selector新注册channel。

### 总结
jetty基于nio处理连接的过程总结为如下图：

![jetty nio](/assets/images/jetty_nio.png)

### 参考
1、[Jetty Architecture](http://www.eclipse.org/jetty/documentation/current/architecture.html#basic-architecture)

2、[Jetty 的工作原理以及与 Tomcat 的比较](http://www.ibm.com/developerworks/cn/java/j-lo-jetty/)
