---
title: Disruptor 学习与使用
date: 2017-01-11 00:01:32
author: "acyouzi"
# cdn: header-off
# header-img: img/docker.png
tags:
	- 并发
	- Disruptor
---

### Disruptor 基本使用

    public class Main {
        static public class MyEvent {
            private int val;
            public int getVal() {
                return val;
            }
            public void setVal(int val) {
                this.val = val;
            }
        }
        public static void main(String[] args) throws InterruptedException {
            Disruptor<MyEvent> disruptor = new Disruptor<MyEvent>(MyEvent::new, 1024, r -> {
                return new Thread(r);
            },ProducerType.SINGLE,new YieldingWaitStrategy());
            disruptor.handleEventsWith((event, sequence, endOfBatch) -> System.out.println(event.getVal() +" -- "+ sequence +" -- "+ endOfBatch ));
            disruptor.start();
           
            //发布事件
            RingBuffer<MyEvent> ringBuffer = disruptor.getRingBuffer();
            for(int l = 0; l<100; l++){
                // 1. 获取下一个可用事件槽
                long seq = ringBuffer.next();
                // 2. 获取事件引用
                MyEvent event = ringBuffer.get(seq);
                event.setVal(l);
                // 3. 发布事件
                ringBuffer.publish(seq);
                Thread.sleep(1000);
            }
            // 阻塞，直到所有消息处理完
            disruptor.shutdown();
        }
    }

1. 创建 Disruptor 对象时的参数 ProducerType 其中 ProducerType.SINGLE 代表只有一个生产者，ProducerType.MULTI 代表有多个生产者
2. WaitStrategy 时等待的策略，不同的策略有不同的性能
    * BlockingWaitStrategy 使用重入锁实现阻塞,一旦不满足条件直接挂起。是最低效的策略，但其对CPU的消耗最小并且在各种不同部署环境中能提供更加一致的性能表现
    * SleepingWaitStrategy 进行多次尝试获取，当尝试达到一定次数以后 yield 或者 LockSupport 挂起，性能表现跟BlockingWaitStrategy差不多，对CPU的消耗也类似，但其对生产者线程的影响最小，适合用于异步日志类似的场景
    * YieldingWaitStrategy 尝试多次，次数可以设置，当达到一定次数，调用 yield，性能是最好的，适合用于低延迟的系统。在要求极高性能且事件处理线数小于CPU逻辑核心数的场景中，推荐使用此策略；例如，CPU开启超线程的特性    
    * 还有几个策略，可以直接看官方文档 [官方文档](https://lmax-exchange.github.io/disruptor/docs/index.html)
3. 注意发布事件的步骤，ringBuffer.publishEvent 是对上述发布事件步骤的封装

### EventProcessor 与 WorkerPool
这是两种消息消费的模式，Disruptor 实例通过 handleEventsWith 设置使用 EventProcessor，通过 handleEventsWithWorkerPool 设置使用 WorkerPool

每一个消息都要经过使用 EventProcessor 注册的所有消费者处理，同时使用 EventProcessor 可以设置一些复杂的消费顺序，如一个消息先经过 A 和 B 消费者，在 B 消费者执行完后执行 C 消费者，在 A 和 C 消费者执行完后执行 D 消费者，见下面的例子

    public class Main {
        public static void main(String[] args) throws InterruptedException {
            Disruptor<MyEvent> disruptor = new Disruptor<MyEvent>(MyEvent::new,1024, r -> {
                return new Thread(r);
            }, ProducerType.SINGLE,new YieldingWaitStrategy());

            /**
            *    A ---------> C
            *                 |
            *    B ---------> D
            * */
            EventHandler<MyEvent> h_a = (event, sequence, endOfBatch) -> System.out.println("A handler --- " + event.getVal() + " -- " + sequence + " -- " + endOfBatch);
            EventHandler<MyEvent> h_b = (event, sequence, endOfBatch) -> System.out.println("B handler --- " + event.getVal() + " -- " + sequence + " -- " + endOfBatch);
            EventHandler<MyEvent> h_c = (event, sequence, endOfBatch) -> System.out.println("C handler --- " + event.getVal() + " -- " + sequence + " -- " + endOfBatch);
            EventHandler<MyEvent> h_d = (event, sequence, endOfBatch) -> System.out.println("D handler --- " + event.getVal() + " -- " + sequence + " -- " + endOfBatch);

            disruptor.handleEventsWith(h_a,h_b);
            disruptor.after(h_a).handleEventsWith(h_c);
            disruptor.after(h_b,h_c).handleEventsWith(h_d);
            disruptor.start();
            RingBuffer<MyEvent> ringbuffer = disruptor.getRingBuffer();
            // 发布者
            for (int i = 0; i < 10; i++) {
                long seq = ringbuffer.next();
                ringbuffer.get(seq).setVal(i);
                ringbuffer.publish(seq);
                Thread.sleep(1000);
            }
            disruptor.shutdown();
        }
    }

在使用 WorkerPool 时，消息只会被多个消费者中的某一个消费。示例如下

    public class Main {
        public static void main(String[] args) throws InterruptedException {
            Disruptor<MyEvent> disruptor = new Disruptor<MyEvent>(MyEvent::new,1024, r -> {
                return new Thread(r);
            }, ProducerType.SINGLE,new YieldingWaitStrategy());

            WorkHandler<MyEvent> h_1 = event -> System.out.println(Thread.currentThread().getName() + event.getVal());
            WorkHandler<MyEvent> h_2 = event -> System.out.println(Thread.currentThread().getName() + event.getVal());
            disruptor.handleEventsWithWorkerPool(h_1,h_2);
            disruptor.start();
            RingBuffer<MyEvent> ringbuffer = disruptor.getRingBuffer();
            for (int i = 0; i < 10; i++) {
                long seq = ringbuffer.next();
                ringbuffer.get(seq).setVal(i);
                ringbuffer.publish(seq);
                Thread.sleep(1000);
            }
            disruptor.shutdown();
        }
    }

### 直接使用 RingBuffer
在一些简单的应用场景下可以直接使用 RingBuffer 而不是 Disruptor 对象。 RingBuffer 可以直接配合 EventProcessor 或者 WorkerPool 使用，下面是配合 EventProcessor 的例子 

    public class Main {
        public static void main(String[] args) throws InterruptedException {
            final RingBuffer<Message> ringBuffer = RingBuffer.createSingleProducer(new EventFactory<Message>() {
                public Message newInstance() {
                    return new Message();
                }
            },1024,new YieldingWaitStrategy());

            SequenceBarrier barrier = ringBuffer.newBarrier();
            final BatchEventProcessor<Message> processor = new BatchEventProcessor<Message>(ringBuffer, barrier, new EventHandler<Message>() {
                public void onEvent(Message event, long sequence, boolean endOfBatch) throws Exception {
                    System.out.println(event.getName() + " -- " + sequence + " -- " + endOfBatch);
                }
            });
            ringBuffer.addGatingSequences(processor.getSequence());
            new Thread(processor).start();
            long seq;
            for( int i = 1 ; i < 10 ;i++){
                seq = ringBuffer.next();
                ringBuffer.get(seq).setName("Test");
                ringBuffer.publish(seq);
                Thread.sleep(1000);
            }
            processor.halt();
        }
    } 

RingBuffer 配合 WorkerPool 使用的例子

    public class Main {
        public static void main(String[] args) throws InterruptedException {
            final RingBuffer<Message> ringBuffer = RingBuffer.createSingleProducer(new EventFactory<Message>() {
                public Message newInstance() {
                    return new Message();
                }
            },1024,new YieldingWaitStrategy());

            SequenceBarrier barrier = ringBuffer.newBarrier();
            WorkerPool<Message> workerPool = new WorkerPool<Message>(ringBuffer, barrier, new ExceptionHandler<Message>() {
                public void handleEventException(Throwable ex, long sequence, Message event) {ex.printStackTrace();}
                public void handleOnStartException(Throwable ex) {}
                public void handleOnShutdownException(Throwable ex) {}
            }, new WorkHandler<Message>() {
                public void onEvent(Message event) throws Exception {
                    System.out.println(event.getName());
                }
            });
            workerPool.start(Executors.newFixedThreadPool(5));

            long seq;
            for( int i = 1 ; i < 10 ;i++){
                seq = ringBuffer.next();
                ringBuffer.get(seq).setName("Test");
                ringBuffer.publish(seq);
                Thread.sleep(1000);
            }
        }
    }

### Disruptor 多个生产者多个消费者示例
首先需要编写 Consumer 类实现 WorkHandler 接口

    public class Consumer implements WorkHandler<MyEvent> {
        @Override
        public void onEvent(MyEvent event) throws Exception {
            System.out.println(Thread.currentThread().getName() +" ---- "+ event.getVal());
        }
    }

其中 MyEvent 类如下
    
    public class MyEvent {
        private int val;

        public int getVal() {
            return val;
        }

        public void setVal(int val) {
            this.val = val;
        }
    }

异常处理函数 MyException 的代码
    
    public class MyException implements ExceptionHandler<MyEvent>{
        public void handleEventException(Throwable ex, long sequence, MyEvent event) {
            ex.printStackTrace();
        }
        public void handleOnStartException(Throwable ex) {
            ex.printStackTrace();
        }
        public void handleOnShutdownException(Throwable ex) {
            ex.printStackTrace();
        }
    }

下面是主要业务逻辑

    public class Main {
        public static void main(String[] args) {
            // 注意生产者类型是 ProducerType.MULTI
            RingBuffer<MyEvent> ringBuffer = RingBuffer.create(ProducerType.MULTI,MyEvent::new,1024,new YieldingWaitStrategy());
            SequenceBarrier barrier = ringBuffer.newBarrier();
            List<Consumer> list = Stream.generate(Consumer::new).limit(4).collect(Collectors.toList());
            Consumer[] events = new Consumer[0];
            events = list.toArray(events);
            WorkerPool<MyEvent> workerPool = new WorkerPool<MyEvent>(ringBuffer, ringBuffer.newBarrier(), new MyException(), events);

            ringBuffer.addGatingSequences(workerPool.getWorkerSequences());
            workerPool.start(Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors()));

            List<Thread> threads = Stream.generate(()->new Thread( ()->{
                for (int i = 0; i < 5; i++) {
                    System.out.println("producer : " + Thread.currentThread().getName());
                    long seq = ringBuffer.next();
                    ringBuffer.get(seq).setVal(i);
                    ringBuffer.publish(seq);
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            })).limit(3).collect(Collectors.toList());
            threads.stream().forEach(thread -> thread.start());
            threads.stream().forEach(thread -> {
                try {
                    thread.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
    }

1. 使用 ProducerType.MULTI 创建 RingBuffer
2. 创建多个消费者，添加到 WorkerPool 中 
3. 把门控序列添加的 RingBuffer 中， ringBuffer.addGatingSequences(workerPool.getWorkerSequences())
4. 创建多个生产者线程

### 其他
1. RingBuffer 是一个特殊的环形队列，这个网上有很多介绍，这个环形队列特殊在没有尾指针，只维护了一个指向下一个可用位置的序号
2. RingBuffer 大小必须是 2 的指数倍
3. 缓存填充，链接： [神奇的缓存填充]( http://ifeve.com/disruptor-cacheline-padding/ )
        
        // cpu 三级缓存都是按行为单位统一刷写，每行大小 64B
        // 为避免缓存失效造成的性能损失，使用不会被修改变量填充满一行缓存行
        public long p1, p2, p3, p4, p5, p6, p7; // cache line padding
        private volatile long cursor = INITIAL_CURSOR_VALUE;
        public long p8, p9, p10, p11, p12, p13, p14; // cache line padding