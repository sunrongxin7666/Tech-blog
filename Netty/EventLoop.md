# Netty源码分析-3:  EventLoop 接口和线程模型
@(Java)[netty]
线程池+等待队列 避免重新创建线程，但还是有线程切换的代价，创建管理线程也是有代价的；

事件循环==EventLoop

```
    public static void executeTaskInEventLoop() {
        boolean terminated = true;
        //...
        while (!terminated) {
            List<Runnable> readyEvents = blockUntilEventsReady();
            for (Runnable ev: readyEvents) {
                ev.run();
            }
        }
    }
    private static final List<Runnable> blockUntilEventsReady() {
        return Collections.<Runnable>singletonList(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
    }
```
EventLoop有两部分构成：并发+网络框架
- 并发：io.netty.util.concurrent构建在java.util.concurrent基础上；
- 网络：io.netty.channel

EventLoop (I)
- -> EventExecutor(I) 并发
- -> EventLoopGroup(I) 网络
- 以上二者都继承 -> EventExecutorGroup 其本质是一种 `ScheduleExecutorService`

```
public interface EventLoop extends OrderedEventExecutor, EventLoopGroup {
    @Override
    EventLoopGroup parent();
}
```
每一个EventLoop都会分配一个Thread，该EventLoop中的所有IO操作都是由该Thread来处理，避免上线文切换。一个EventLoop可能处理多个Channel上的事件；

因为Netty的线程模型中，可以确定当前Channel和Thread的绑定关系：
- 存在绑定关系，则立刻执行；
- 如果不存在，则将事件放入对应Thread的等待队列中（每个EventLoop中都有自己等待队列），等待合适时机去执行

因此Channel和Thread的直接交互，无需ChannelHandler额外的同步操作，不拥塞线程，不线程切换。

EventLoop线程分配：
- NIO：EventLoopGroup含有若干个EventLoop，每个EventLoop指定一个Thread来处理N个Channel的事件；
- OIO：EventLoopGroup含有若干个EventLoop，每个EventLoop指定一个Thread来处理1个Channel的事件；

