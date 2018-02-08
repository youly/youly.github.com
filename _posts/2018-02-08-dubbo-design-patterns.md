---
layout: post
title: dubbo框架中一些值得学习的设计模式
category: 设计
tag: [dubbo]
---

微服务已经是一种应用很广的架构模式了，而微服务框架中比较常用的有spring could、dubbo，想要熟悉并应用微服务架构，可以从这二者中选一。最近排查问题，重新看了下dubbo框架相关源码，总结下其中一些值得学习的设计模式。

### SPI 模式
SPI（Service Provider Interface），一种基于接口的编程模式，由协议或者框架开发者制定了基本接口，厂商基于约定的接口去实现具体的服务。程序运行时使用哪个服务，由配置决定。[SPI](https://docs.oracle.com/javase/tutorial/sound/SPI-intro.html) 机制来源于jkd 1.6，[dubbo](https://github.com/alibaba/dubbo/blob/master/dubbo-common/src/main/java/com/alibaba/dubbo/common/extension/ExtensionLoader.java)改进了SPI 机制，实现类按需实例化，而不是启动时就实例化所有实现类。service provider在dubbo中称作extension。

    // 加载扩展点配置
    private Map<String, Class<?>> loadExtensionClasses() {
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        if (defaultAnnotation != null) {
            String value = defaultAnnotation.value();
            if (value != null && (value = value.trim()).length() > 0) {
                String[] names = NAME_SEPARATOR.split(value);
                if (names.length > 1) {
                    throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                                + ": " + Arrays.toString(names));
                }
                if (names.length == 1) cachedDefaultName = names[0];
            }
        }
    
        Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
        // 从类路径配置文件中加载扩展点配置至内存
        loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
        loadFile(extensionClasses, DUBBO_DIRECTORY);
        loadFile(extensionClasses, SERVICES_DIRECTORY);
        return extensionClasses;
    }

dubbo中扩展配置以service_name=service_provider的方式存储于文件中(dubbo/internal目录)，运行时通过预先配置的service_name去查找对应的实现类。使用hashmap方式存储映射，避免了使用if-else式的语句。

    // 按需实例化指定扩展点
    public T getExtension(String name) {
        if (name == null || name.length() == 0)
            throw new IllegalArgumentException("Extension name == null");
        if ("true".equals(name)) {
            return getDefaultExtension();
        }
        Holder<Object> holder = cachedInstances.get(name);
        if (holder == null) {
            cachedInstances.putIfAbsent(name, new Holder<Object>());
            holder = cachedInstances.get(name);
        }
        Object instance = holder.get();
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    // 真正去创建扩展点
                    instance = createExtension(name);
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }

有时候配置不是写死在文件或者环境变量中，而是放在内存中且用户动态可配置。dubbo中配置以URL参数的方式表示，如service_provider?configA=xxx&configB=xxx。运行时从URL中提取配置参数来实例化对应的实现类并调用该类的方法。例如dubbo生成远程服务代理时，有如下代码：

    private static final Protocol refprotocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
    URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
    // 生成某个协议的服务代理，具体是哪个协议，由url中的参数决定，而url中的参数可动态配置。
    // 这里 refprotocol 是Protocol接口的一个特殊实现类，封装了根据参数调用其他实现类的功能。
	invoker = refprotocol.refer(interfaceClass, url);
	

### AOP 模式
AOP（Aspect Orient Programming），作为面向对象编程的一种补充，广泛应用于处理一些具有横切性质的与业务功能无关的管理服务，如事务管理、安全检查、缓存、对象池管理等。dubbo通过接口实现类的方式巧妙地实现了AOP的功能。还是以Protocal.class为例，看配置文件[com.alibaba.dubbo.rpc.Protocol
](https://github.com/alibaba/dubbo/blob/master/dubbo-rpc/dubbo-rpc-api/src/main/resources/META-INF/dubbo/internal/com.alibaba.dubbo.rpc.Protocol)，Protocol有多个实现类：

    filter=com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper
    listener=com.alibaba.dubbo.rpc.protocol.ProtocolListenerWrapper
    mock=com.alibaba.dubbo.rpc.support.MockProtocol

其中[ProtocolListenerWrapper](https://github.com/alibaba/dubbo/blob/master/dubbo-rpc/dubbo-rpc-api/src/main/java/com/alibaba/dubbo/rpc/protocol/ProtocolListenerWrapper.java)并不是一个真正的具体协议实现，而是对Protocol的wrapper封装，看下其实现（只列了关键部分，其他无关的已省略）：

    public class ProtocolListenerWrapper implements Protocol {
    
        private final Protocol protocol;
    
        public ProtocolListenerWrapper(Protocol protocol) {
            if (protocol == null) {
                throw new IllegalArgumentException("protocol == null");
            }
            this.protocol = protocol;
        }
        
        public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
            if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
                return protocol.export(invoker);
            }
            return new ListenerExporterWrapper<T>(protocol.export(invoker),
                    Collections.unmodifiableList(ExtensionLoader.getExtensionLoader(ExporterListener.class)
                            .getActivateExtension(invoker.getUrl(), Constants.EXPORTER_LISTENER_KEY)));
        }
        
        // 代码省略
    
    }

可以看到ProtocolListenerWrapper调用完具体协议后，还做了其他一些功能。那么框架是怎么实现把具体Protocol实现类注入到ProtocolListenerWrapper呢？通过构造函数。ExtensionLoader在加载扩展点实现类时，会判断当前扩展点是否包含具有扩展点接口类型参数的构建函数，如下代码：

    private void loadFile(Map<String, Class<?>> extensionClasses, String dir) {
        // 代码省略
        try {
            // 判断是否包含具有type参数类型的构造函数，如果包含，则认为是一个wapper类，先存起来，后面实例化扩展点时会用到
            clazz.getConstructor(type);
            Set<Class<?>> wrappers = cachedWrapperClasses;
            if (wrappers == null) {
                cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
                wrappers = cachedWrapperClasses;
            }
            wrappers.add(clazz);
        } catch (NoSuchMethodException e) {
        }
        // 代码省略
    }

创建扩展实现类时的代码：

    private T createExtension(String name) {
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            injectExtension(instance);
            // 返回的实际上是经过wrap的类，而不是原始的类
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (wrapperClasses != null && wrapperClasses.size() > 0) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                    type + ")  could not be instantiated: " + t.getMessage(), t);
        }
    }
    
需要注意的是，当一个类有个多wrapper实现时，这些wrapper的调用是无序的，因为 cachedWrapperClasses 使用的是 ConcurrentHashSet。

### IOC 模式
IOC (Inversion of Control)，依赖由框架注入，无需自己管理。dubbo中的扩展点，有可能会依赖其他扩展点。dubbo扩展点本身是dubbo自己管理动态编译生成的，无法很方便的与spring context继承。那么其注入怎么实现注入的呢？

    private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
                for (Method method : instance.getClass().getMethods()) {
                    if (method.getName().startsWith("set")
                            && method.getParameterTypes().length == 1
                            && Modifier.isPublic(method.getModifiers())) {
                        Class<?> pt = method.getParameterTypes()[0];
                        try {
                            String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                            // 获取依赖
                            Object object = objectFactory.getExtension(pt, property);
                            if (object != null) {
                                // 依赖注入
                                method.invoke(instance, object);
                            }
                        } catch (Exception e) {
                            logger.error("fail to inject via method " + method.getName()
                                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
    
可以看到，dubbo通过反射的方式，判断方法是否以set开头，如果是则认为是一个依赖，通过objectFactory获取依赖实现并注入。

这里只是实现了简单的依赖注入，并没有处理循环依赖的问题。

### 总结
dubbo框架通过接口实现的方式实现了动态SPI，AOP功能。定义一个接口，可以有如下实现：
    
    interface，接口定义
    - interfaceImpl1，具体功能实现类
    - interfaceImpl2，具体功能实现类
    - interfaceImplN，具体功能实现类
    - interfaceWrapperImpl，AOP实现类
    - interfaceAdaptiveImpl，动态SPI实现类





