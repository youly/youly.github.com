---
layout: post
title: jvm本地缓存的实现
category: 设计
tags: [java, cache]
---

### 是remotecache，而不是localcache

今日做微信支付接入时有个问题是，每次请求生成预付款订单时都得带上一个access_token。虽然不知道这个access_token是否真的有必要，但这种做法与支付宝支付接入相比，明显是增加了麻烦。根据微信的文档，获得的这个access_token是有个过期时间的，意思是开发者得自己缓存这个token，否则每次去调用刷新将增加额外的网络开销。

那么如何缓存这个token呢？当前我们使用的框架实现了localcache，只要继承这个localcache就可以在jvm内存中缓存这个token，每次客户端请求支付时直接从内存中取，取出来的时候先判断下是否过期，如果过期才去微信接口重新获取再缓存。想起来是个不错的解决方案，于是马上写代码实现之。写完之后细想，我们的web程序是部署在多台机器上通过nginx分发流量的，即使用了多个jvm，每个jvm都有自己的localcache，客户端请求支付时都会去调用微信接口获取access_token。而文档里又写着，每调用一次，之前获取的将token失效。这搞毛线，jvm之间互相使对方失效，完美的cache计划泡汤了…

### 说说localcache

上面的问题有很多解决方案，比如使用redis集中式cache、dbcache等。但接下来要讲的是localcache，毕竟这个是今天遇到的一个坑。

localcache相对于集中式cache有几个优点，比如：

1、直接内存访问，速度快，没有网络开销

2、便于管理，没有数据同步的烦恼

通常在业务数据不是很敏感或者敏感度低的时候，很适合使用localcahce。比如我们现在的首页，商品基本上是每天十点加载并缓存，之后很少就会有更新了。即使有更新，延迟一点点刷新也没有问题。

### localcache的实现

实现一个localcache，实现以下三个对象就可以：

1、LocalCacheWraper，与具体业务关联的对象，提供访问localcache的接口，并封装一些业务逻辑，如判断缓存是否已失效等。

2、AbstractLocalCache，提供对象cache，从其他数据源加载数据，能再高并发的场景下原子性的get，load，reload，

3、LocalCacheManager，管理对象cache，负责定时reload缓存、接收外部消息更新缓存

上代码，AbstractLocalCache实现

    import java.util.concurrent.atomic.AtomicBoolean;
    import java.util.concurrent.atomic.AtomicReference;

    public abstract class AbstractLocalCache<T> {

        private final AtomicReference<T> objectRef;

        private final AtomicBoolean isLoad;

        public AbstractLocalCache() {
            super();
            this.objectRef = new AtomicReference<>();
            this.isLoad = new AtomicBoolean(false);
            if (!isLazy()) {
                init();
            }
        }

        /**
         * 缓存初始化
         */
        protected void init() {
            if (isLoad.get()) {
                return;
            }

            synchronized (init) {
                if (isLoad.get()) {
                    return;
                }
                T object = load();
                this.objectRef.set(object);
                LocalCacheManager.getInstance().register(this);
                isLoad.set(true);
            }
        }

        /**
         * 收到数据更新广播，重新加载数据
         */
        public void onMessage(Object message) {
            reload();
        }

        /**
         * 缓存初始化
         */
        public T get() {
            init();
            return objectRef.get();
        }

        /**
         * 重新加载缓存
         */
        public void reload() {
            T object = load();
            this.objectRef.set(object);
        }

        /**
         * 定时刷新间隔，由子类实现
         */
        abstract public long getReloadPeriod();

        /**
         * 是否延迟加载，由子类实现
         */
        abstract public boolean isLazy();

        /**
         * 从数据源加载缓存，由子类实现
         */
        abstract public boolean load();

        /**
         * 订阅消息key，由子类实现
         */
        abstract public boolean key();
    }

LocalCacheManager实现：

    import java.util.concurrent.ConcurrentHashMap;
    import java.util.concurrent.Executors;
    import java.util.concurrent.ScheduledExecutorService;
    import java.util.concurrent.TimeUnit;

    public class LocalCacheManager {

        private static final LocalCacheManager INSTANCE = new LocalCacheManager();

        public static LocalCacheManager getInstance() {
            return INSTANCE;
        }

        private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(8);

        private final ConcurrentHashMap<String, LocalCache<?>> localCacheMap = new ConcurrentHashMap<>();

        private LocalCacheManager() {
        }

        /**
         * 注册cache
         */
        public void register(LocalCache<?> localCache) {

            localCacheMap.putIfAbsent(localCache.key(), localCache);
            scheduleReload(localCache);
            scribeReloadMessage(localCache);
        }


        /**
         * 线程池固定间隔调度，刷新cache
         */
        public void scheduleReload(final LocalCache<?> localCache) {
            scheduler.scheduleWithFixedDelay(new Runnable() {

                @Override
                public void run() {
                    localCache.reload();
                }
            }, localCache.getReloadPeriod(), localCache.getReloadPeriod(), TimeUnit.MILLISECONDS);
        }

        /**
         * 基于zookeeper实现的发布订阅服务，跨jvm之间同步缓存
         */
        public void scribeReloadMessage(final LocalCache<?> localCache) {
            ZookeeperPubSubService.getInstance().register(localCache.key(), localCache);
        }
    }

上面的manager类实现了多jvm之间同步cache，其实可以解决最初的问题，但localcache毕竟不是为适用这种场景而设计的，远没有remotecache可靠。

### 参考
1、[Local Cache的小TIP](http://blog.csdn.net/cenwenchu79/article/details/6076513)


