---
layout    : post
title     : "ONOS 事件子系统"
date      : 2023-06-30
lastupdate: 2023-06-30
categories: Others
---

<p align="center"><img src="/assets/img/public/onos.png"></p>

本文先介绍ONOS事件子系统的实现原理, 再介绍如何在自己的服务中利用它实现业务的解耦。

## 为什么需要它

ONOS的事件子系统可以理解为**<mark>消息总线</mark>**，各个服务都可以利用它提供的**<mark>订阅-发布</mark>**特性与其他服务进行消息通信。

假如没有这套机制，比如服务A想通知服务B一个消息。A就需要显式地调用B暴露的接口, 并且这个过程还只能在A的上下文执行，且它还是同步而不是异步的。

这还只是两个服务的情况，这时再加入一个服务C也想收到这个消息，就需要修改服务A的实现，这实在不优雅。

## 实现原理

### Event 与 Event Sink

ONOS 中有很多现成的 Event 类型, 如 `DeviceEvent`、`MastershipEvent`、`ClusterEvent`, 我们所说的发布或接收事件就是实例化这些类进行操作

<p align="center"><img src="/assets/img/p4-onos-event/pic1-m.png"></p>

Event Sink (事件槽) 接口则表示响应特定 Event 类型的 handler. Event Sink 的类型与 Event 一一对应。Event Sink 是由事件**发布者**管理.

举个例子, 管理设备的 `DeviceManager` 负责 `DeviceEvent`，它会有一个 listener registry 注册表，它记录了其他哪些服务对 `DeviceEvent` 感兴趣.

<p align="center"><img src="/assets/img/p4-onos-event/pic2-m.png"></p>

### 发布事件

当某服务需要发布事件时，需要调用父类 `AbstractListenerManager` 的 `post()` 接口, 这会将事件转交给 `EventDeliveryService`

```java
class AbstractListenerManager<E extends Event, L extends EventListener<E>>
    implements ListenerService<E, L> {
    ...
    @Reference(cardinality = ReferenceCardinality.MANDATORY)
    protected EventDeliveryService eventDispatcher;
    
    protected void post(E event) {
        if (event != null && eventDispatcher != null) {
            eventDispatcher.post(event);
        }
    }
```

`EventDeliveryService`在ONOS中的实现是`CoreEventDispatcher`, 其 UML 关系如下图所示 

(重点关注`CoreEventDispatcher`和其父类`DefaultEventSinkRegistry`和其内部类`DispatchLoop`)

<p align="center"><img src="/assets/img/p4-onos-event/pic3-m.png"></p>

`CoreEventDispatcher`是`Event`的交通枢纽, ONOS 系统中的所有`Event`都会在这里分发. 

`CoreEventDispatcher`定义了3个 `DispatchLoop`, 每个`DispatchLoop`运行在不同的线程. 

不同的`Event`类型在不同的`DispatchLoop`进行 分发, 这个映射关系是固定写的. 

如`DeviceEvent`对应`topologyDispatcher`...没有出现在这里的`Event`将会使用默认的`defaultDispatcher`

```java
public class CoreEventDispatcher extends DefaultEventSinkRegistry
        implements EventDeliveryService {
    ...
    
    private DispatchLoop topologyDispatcher = new DispatchLoop("topology");
    private DispatchLoop programmingDispatcher = new DispatchLoop("programming");
    private DispatchLoop defaultDispatcher = new DispatchLoop("default");

    private Map<Class, DispatchLoop> dispatcherMap =
            new ImmutableMap.Builder<Class, DispatchLoop>()
                .put(TopologyEvent.class, topologyDispatcher)
                .put(DeviceEvent.class, topologyDispatcher)
                .put(LinkEvent.class, topologyDispatcher)
                .put(HostEvent.class, topologyDispatcher)
                .put(FlowRuleEvent.class, programmingDispatcher)
                .put(IntentEvent.class, programmingDispatcher)
                .build();

    private Set<DispatchLoop> dispatchers =
            new ImmutableSet.Builder<DispatchLoop>()
                .addAll(dispatcherMap.values())
                .add(defaultDispatcher)
                .build();

```

服务在发布事件时, 会调用`eventDispatcher.post(event)`. 这里可以看出，它就是将事件放到该类型事件对应的`DispatchLoop`的队列中.

```java
public void post(Event event) {
        if (!getDispatcher(event).add(event)) {
            log.error("Unable to post event {}", event);
        }
    }
```

而在 `DispatchLoop` 自身的线程上下文, 就是不停地从队列中取出事件, 再找到该类型对应的已注册的 `EventSink` 进行 `process()`

```java
public void run() {
    log.info("Dispatch loop({}) initiated", name);
    while (!stopped) {
        try {
            // Fetch the next event and if it is the kill-pill, bail
            Event event = eventsQueue.take();
            if (event != KILL_PILL) {
                process(event);
            }
        } catch (InterruptedException e) {
            log.warn("Dispatch loop interrupted");
        } catch (Exception | Error e) {
            log.warn("Error encountered while dispatching event:", e);
        }
    }
    log.info("Dispatch loop({}) terminated", name);
}

private void process(Event event) {
    EventSink sink = getSink(event.getClass());
    if (sink != null) {
        lastSink = sink;
        stopwatch.start();
        sink.process(event);
        stopwatch.reset();
    } else {
        log.warn("No sink registered for event class {}",
                 event.getClass().getName());
    }
}
```

### 注册 Event Sink

事件的发布者需要提前向`EventDeliveryService`注册`Event Sink`

如`DeviceManager.java`中的`eventDispatcher.addSink(DeviceEvent.class, listenerRegistry)`
如`MastershipManager.java`中的`eventDispatcher.addSink(MastershipEvent.class, listenerRegistry)`

### Event Sink 的 process

这里的逻辑很简单, 就是遍历所有订阅了这个事件的 `listener`, 调用它们各自的`event()`方法 
```java
public class ListenerRegistry<E extends Event, L extends EventListener<E>>
        implements ListenerService<E, L>, EventSink<E> {
        
    ...
    public void process(E event) {
            for (L listener : listeners) {
                try {
                    lastListener = listener;
                    lastStart = System.currentTimeMillis();
                    if (listener.isRelevant(event)) {
                        listener.event(event);
                    }
                    lastStart = 0;
                } catch (Exception error) {
                    reportProblem(event, error);
                }
            }
        }
```

### 事件的监听

事件的监听分为两部分: 

- 发布者提供注册监听的接口
- 订阅者需要调用该接口

下图表示服务A `(A_Manager)`监听了服务B`(B_Manager)`的事件.
事件发布者 B 通过继承 `AbstractListenerManager`, 实现了 `addListner` 等接口, 再通过 `B_Service` 对外暴露这些接口

<p align="center"><img src="/assets/img/p4-onos-event/pic4-m.png"></p>

这样, 服务A便可以调用服务B暴露的接口向服务B注册 listener
```java
@Reference(cardinality = ReferenceCardinality.MANDATORY)
public B_Service b_Service;

A_listen_B_Listener listener = new A_listen_B_Listener();
b_Service.addListener(listener);
```

## 应用

下面是一个更加完善的例子: 服务B对外发布 `EVENT_B_X` 事件, 并携带 B_Extra 类型的额外数据, 服务A监听这些事件进行不同的处理.

定义 `B_Event` 事件
```java
public class B_Event extends AbstractEvent<B_Event.Type, B_Extra> {

    public enum Type {
        EVENT_B_1,
        EVENT_B_2,
        EVENT_B_3
    }

    public B_Event(Type type, B_Extra extra) {
        super(type, extra);
    }
    ...
}
```

定义 `B_Listener` 接口
```java
public interface B_Listener extends EventListener<B_Event> {
}
```

让 `B_Service`, 继承 `ListenerService`
```java
public interface OvsService extends ListenerService<B_Event, B_Listener>{
    ...
}

```

服务B中在需要的位置加入发布事件

```java
public class B_Manager extends AbstractListenerManager<B_Event, B_Listener> 
    implements B_Service
{
    ...
    post(new B_Event(EVENT_B_1, extra));
}
```

服务A 实现服务B 相关事件的处理逻辑.
```java
public class A_Manager implements A_Service {
    
    @Reference(cardinality = ReferenceCardinality.MANDATORY)
    public B_Service b_Service;
    
    private final A_listen_B_Listener listener = new A_listen_B_Listener();
    
    b_Service.addListener(listener);
    
    private class A_listen_B_Listener implements B_Listener {
        @Override
        public void event(EVENT_B_1 event) {
            B_Extra extra = event.subject();
            switch (event.type()) {
                case EVENT_B_1:
                    ...
                case EVENT_B_2:
                    ...
                case EVENT_B_3:
                    ...
                default:
                    break;
            }
        }
    }
}
```

