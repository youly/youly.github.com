---
layout: post
title: 使用zookeeper实现动态配置管理
category: 设计
tags: [zookeeper, config]
---

###目标
1、在内存中存储一套配置，业务逻辑每次从JVM本地内存中读取配置然后执行相应逻辑

2、修改配置，不重启JVM，业务逻辑下次读取配置时能感知到变化

3、多个JVM能共享这套配置

###实现
实现上述目标，可以把配置写在数据库中，每次修改配置，更新数据表相应字段即可。但若业务需频繁读取此配置，则会增加额外的性能开销。

利用ZooKeeper的watcher机制能很好的实现上述目标：在zookeep中新建配置根节点，在根节点下初始化配置。虽然zookeeper支持类似文件系统的目录结构，但实现这套配置管理，只需支持根节点下面一层。每次有配置变更，重新加载所有配置，以减少内存中数据与zookeeper中数据不一致的可能。

代码主要分成两部分：

1、处理zookeeper连接断开、数据更新等事件通知，保证与zookeeper的连接始终有效：

    @Override
    public void process(WatchedEvent event) {

        log.debug("received event:" + event.toString());

        switch (event.getType()) {
            case None:
                switch (event.getState()) {
                    case Expired:
                        log.debug("expired:" + this.zooKeeper.getSessionId());
                        try {
                            this.zooKeeper.close();
                            this.zooKeeper = createZooKeeper();
                            this.monitor();
                        } catch (Exception e) {
                            throw new RuntimeException(e);
                        }
                        break;
                    case Disconnected:
                        log.debug("disconnected:" + this.zooKeeper.getSessionId());
                        try {
                            this.zooKeeper.close();
                            this.zooKeeper = createZooKeeper();
                            this.monitor();
                        } catch (Exception e) {
                            throw new RuntimeException(e);
                        }
                        break;
                    case SyncConnected:
                        countDownLatch.countDown();
                        break;
                }
                break;
            case NodeChildrenChanged:
                //节点删除、创建同时会促发上层节点NodeChildrenChanged事件，因此可忽略
                //case NodeDeleted:
                //case NodeCreated:
            case NodeDataChanged:
                if (!this.zooKeeper.getState().isAlive()) {
                    try {
                        this.zooKeeper = createZooKeeper();
                    } catch (Exception e) {
                        throw new RuntimeException(e);
                    }
                }
                this.monitor();
                break;
        }
    }

2、加载配置数据并重新注册watcher：

        public void monitor() {
            //更新数据并注册watcher
            try {
                // 监听根节点数据变更、创建、删除
                this.zooKeeper.getData("/root", true, null);
                // 监听子节点数据更新、创建、删除
                List<String> children = this.zooKeeper.getChildren("/root", true);
                for (String child : children) {
                    this.zooKeeper.getData("/root/" + child, true, null);
                }
            } catch (KeeperException e) {
                e.printStackTrace();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

完整代码点[这里](https://github.com/youly/study/blob/master/src/main/java/com/lastww/study/zookeeper/ZookeeperEventContainer.java)

###使用zookeeper需注意的几点

1、收到zookeeper节点数据变更事件后需要重新注册watcher，否则之后不会再收到通知

2、与zookeeper的连接断开后，在session timeout之前可以重新连接

3、节点的版本变更后，即使数据不变，也会收到通知

###参考
1、[ZooKeeper FAQ](http://jm-blog.aliapp.com/?p=1384)

2、https://cwiki.apache.org/confluence/display/ZOOKEEPER/FAQ


