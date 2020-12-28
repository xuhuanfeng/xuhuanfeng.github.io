---
title: Elasticsearch Bulk API学习笔记
date: 2020-12-28 16:27:12
tags: 
  - Elasticsearch
  - Bulk
categories:
  - Elasticsearch
---

# ES Bulk API

在操作ES的过程中，有很多的情况下是需要同时往ES插入很多数据，这个时候有两种方式

第一种是通过ES的单个操作数据接口循环操作数据，这种操作方式会当请求次数多的时候，会有很明显的缺点，请求次数多，容易影响性能；单次插入数据的有效负载低(一个http请求的body携带的实际有效内容)，因此在数据规模比较大的时候，更倾向于使用第二种方式

第二种是通过ES的bulk接口，顾名思义，这个接口是用于批量操作数据的，这个接口本身支持同时使用多种类型的action(插入、删除、更改等)，然后一次提交批量的操作给ES，从而提高网络的有效负载，也一定程度上降低ES的请求数量

<!--more-->

## Bulk API

ES的bulk操作是通过`_bulk`这个endpoint来实现的，方式是`POST`，一次bulk操作可以同时进行不同的操作，语法如下：

```http
POST /INDEX/_bulk
{ "index" : { "_index" : "test", "_id" : "1" } }
{ "field1" : "value1" }
{ "delete" : { "_index" : "test", "_id" : "2" } }
{ "create" : { "_index" : "test", "_id" : "3" } }
{ "field1" : "value3" }
{ "update" : {"_id" : "1", "_index" : "test"} }
{ "doc" : {"field2" : "value2"} }
```

每个操作本身是通过JSON来描述的，操作之间通过`\n`进行分隔，其中的`create`、`index`两个操作均包含一个body，这个body本身也是一个JSON，并且通过`\n`分隔，紧跟在操作描述之后，如果index操作指定了`_index`，则POST的时候，可以不用指定INDEX

注：如果是7.0以下的版本还需要指定`_type:XX`

index示例：

```http
POST /test-20201228/_bulk
{"index": {"_id": "123", "_type" : "_doc"}}
{"name": "xaiver", "age": 25}
{"index": {"_id": "124", "_type": "_doc"}}
{"name": "sharon", "age": 24}
```

其他的根据语法进行操作即可

## Bulk Java API

Java 的bulk本质上是对`_bulk`的封装而已，直接参考下官方的文档即可，简单的示例如下

```java
public class BulkTest {
    public static void main(String[] args) {
        RestHighLevelClient restClient = EsClient.getRestClient();

        BulkRequest bulkRequest = new BulkRequest();
        bulkRequest.add(new IndexRequest("test-20201228", "_doc")
                        .source(XContentType.JSON, "name", "jack", "age", 35));
        bulkRequest.add(new IndexRequest("test-20201228", "_doc")
                        .source(XContentType.JSON, "name", "tony", "age", 50));

        try {
            BulkResponse bulkResponse = restClient.bulk(bulkRequest, RequestOptions.DEFAULT);
            if (bulkResponse.hasFailures()) {
                for (BulkItemResponse bulkItemResponse : bulkResponse) {
                    if (bulkItemResponse.isFailed()) {
                        System.out.println("error....");
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## BulkProcessor

在普通的场景中，通过`bulk`或其对应封装的API来实现即可，然而，在一些特定场景中，单纯使用`bulk`却无法实现，比如说，我们想文档数量达到某一个阈值，或者文档大小达到某一个阈值就提交，避免某一个批次的数据量太大，造成ES处理缓慢，或者当没有达到上面的情况时，根据某个固定的频率提交，避免长时间没有数据提交从而造成活锁的情况

### BulkProcessor使用

根据上面的需求，我们是可以轻松通过实现的，不过，这次这个轮子ES的Java客户端已经造好了，使用方式如下：

```java
public static void main(String[] args) {
    RestHighLevelClient restClient = EsClient.getRestClient();
    
    BulkProcessor.Listener listener = new BulkProcessor.Listener() {

        // 提交之前回调
        @Override
        public void beforeBulk(long executionId, BulkRequest request) {
            System.out.println("executionId: " + executionId);
            System.out.println("before bulk, bulk size: " + request.numberOfActions());
        }

        // 提交之后回调，用于检查成功的、失败的数据
        @Override
        public void afterBulk(long executionId, BulkRequest request, BulkResponse response) {
            System.out.println("after bulk, executionId: " + executionId);
            List<DocWriteRequest<?>> totalRequest = request.requests();

            List<DocWriteRequest<?>> retryList = new ArrayList<>();
            for (BulkItemResponse bulkItem : response.getItems()) {
                if (!bulkItem.isFailed()) {
                    continue;
                }

                // 这里根据具体的业务逻辑来判断是否进行重试操作，其他的逻辑根据情况补充即可
                RestStatus failStatus = bulkItem.getFailure().getStatus();
                switch (failStatus) {
                    case TOO_MANY_REQUESTS:
                    case SERVICE_UNAVAILABLE:
                        // 重试...
                        retryList.add(totalRequest.get(bulkItem.getItemId()));
                        break;
                    default:
                        break;
                }
            }
        }

        // 这里是失败，注意与前面的区别
        @Override
        public void afterBulk(long executionId, BulkRequest request, Throwable failure) {
            System.out.println("失败：" + failure.getMessage());
        }
    };


    BulkProcessor bulkProcessor = BulkProcessor
        .builder(((req, bulkResponseActionListener) ->
            restClient.bulkAsync(req, RequestOptions.DEFAULT, bulkResponseActionListener)), listener)
            // 1000个action提交一次
            .setBulkActions(1000)
            // 1MB提交一次
            .setBulkSize(new ByteSizeValue(1, ByteSizeUnit.MB))
            // 10s提交一次
            .setFlushInterval(TimeValue.timeValueSeconds(10))
            // 并发度
            .setConcurrentRequests(2)
            .build();

}

// 现在只需要将数据塞进来就行了，当到达阈值的时候，会自动进行提交
public void addEvent(BulkProcessor bulkProcessor, IndexRequest request) {
    bulkProcessor.add(request);
}
```

可以看到，通过官方提供的API，我们只需要配置一个`BulkProcessor.Listener`，用于在提交前后获得通知，对应的重试逻辑，以及配置好触发提交的阈值即可，剩下的就只需要添加数据就行，当达到阈值的时候，会自动触发提交

### BulkProcessor实现

大致了解了其使用方式后，接下来看下其实现逻辑，做到知其然知其所以然

`BulkProcessor.Listener`接口就不分析了，其实就是前面提到的三个回调方法而已

#### Builder

BulkProcessor通过Builder模式来提供更加便捷的参数配置模式

BulkProcessor暴露了两个用于创建builder的方法：

```java
public static Builder builder(Client client, Listener listener) {
        Objects.requireNonNull(client, "client");
        Objects.requireNonNull(listener, "listener");
        return new Builder(client::bulk, listener, client.threadPool(), () -> {});
    }

// 我们重点分析这个方法，两者的原理是一样的
public static Builder builder(BiConsumer<BulkRequest, ActionListener<BulkResponse>> consumer, Listener listener) {
    Objects.requireNonNull(consumer, "consumer");
    Objects.requireNonNull(listener, "listener");
    
    // 初始化调度器，底层是通过ScheduledThreadPoolExecutor来提供定时调度的能力
    final ScheduledThreadPoolExecutor scheduledThreadPoolExecutor = Scheduler.initScheduler(Settings.EMPTY);
    
   // 返回Builder实例
    return new Builder(consumer, listener,
                       buildScheduler(scheduledThreadPoolExecutor),
                       () -> Scheduler.terminate(scheduledThreadPoolExecutor, 10, TimeUnit.SECONDS));
}
```

Builder有好几个参数前面已经看过了，接下来完整得看下其他的参数信息

```java
public static class Builder {

    // 消费者，其实就是提交者，这里感觉用“生产者”感觉更加合适一些
    private final BiConsumer<BulkRequest, ActionListener<BulkResponse>> consumer;
    // 监听器
    private final Listener listener;
    // 调度器，核心在这里
    private final Scheduler scheduler;
    // 关闭钩子
    private final Runnable onClose;
    // 并发度，默认是1
    private int concurrentRequests = 1;
    // action的数量，默认是1000
    private int bulkActions = 1000;
    // 数据大小，默认是5MB
    private ByteSizeValue bulkSize = new ByteSizeValue(5, ByteSizeUnit.MB);
    // 定时提交的时间间隔
    private TimeValue flushInterval = null;
    // 回退策略
    private BackoffPolicy backoffPolicy = BackoffPolicy.exponentialBackoff();
    // 索引
    private String globalIndex;
    // 文档类型
    private String globalType;
    // 路由
    private String globalRouting;
    // 流水线
    private String globalPipeline;

    private Builder(BiConsumer<BulkRequest, ActionListener<BulkResponse>> consumer, Listener listener,
                    Scheduler scheduler, Runnable onClose) {
        this.consumer = consumer;
        this.listener = listener;
        this.scheduler = scheduler;
        this.onClose = onClose;
    }
    // 省略一大堆的builder方法

    public BulkProcessor build() {
        // 调用BulkProcessor的构造方法进行初始化
        return new BulkProcessor(consumer, backoffPolicy, listener, concurrentRequests, bulkActions,bulkSize, flushInterval, scheduler, onClose, createBulkRequestWithGlobalDefaults());
    }

    private Supplier<BulkRequest> createBulkRequestWithGlobalDefaults() {
        return () -> new BulkRequest(globalIndex, globalType)
            .pipeline(globalPipeline)
            .routing(globalRouting);
    }
}
```

#### BulkProcessor初始化

```java
BulkProcessor(BiConsumer<BulkRequest, 
              ActionListener<BulkResponse>> consumer, 
              BackoffPolicy backoffPolicy, 
              Listener listener,
              int concurrentRequests, 
              int bulkActions, 
              ByteSizeValue bulkSize, 
              @Nullable TimeValue flushInterval,
              Scheduler scheduler,
              Runnable onClose, 
              Supplier<BulkRequest> bulkRequestSupplier) {
    
    this.bulkActions = bulkActions;
    this.bulkSize = bulkSize.getBytes();
    this.scheduler = scheduler;
    // 获取需要提交的内容
    this.bulkRequest = bulkRequestSupplier.get();
    this.bulkRequestSupplier = bulkRequestSupplier;
    // 注意这个，逻辑的处理核心就在这个类中
    this.bulkRequestHandler = new BulkRequestHandler(consumer, backoffPolicy, listener, scheduler, concurrentRequests);
    // 定时提交策略的实现
    this.cancellableFlushTask = startFlushTask(flushInterval, scheduler);
    this.onClose = onClose;
}
```

#### 添加数据

BulkProcessor提供了很多个重载的add方法，不过最终都会调用`internalAdd`这个方法，重点看下这个方法即可

```java
// 注意这里是加了锁
private synchronized void internalAdd(DocWriteRequest request, @Nullable Object payload) {
    ensureOpen();
    // 添加
    bulkRequest.add(request, payload);
    // 检查是否需要提交
    executeIfNeeded();
}   
```

> executeIfNeeded

```java
private void executeIfNeeded() {
    ensureOpen();
    if (!isOverTheLimit()) {
        return;
    }
    execute();
}
```

> isOverTheLimit

```java
private boolean isOverTheLimit() {
    // 数量满足
    if (bulkActions != -1 && bulkRequest.numberOfActions() >= bulkActions) {
        return true;
    }
    // 大小满足
    if (bulkSize != -1 && bulkRequest.estimatedSizeInBytes() >= bulkSize) {
        return true;
    }
    return false;
}
```

> execute

```java
private void execute() {
    final BulkRequest bulkRequest = this.bulkRequest;
    // 当前的id，其实就是一个单调递增的数值
    final long executionId = executionIdGen.incrementAndGet();

    this.bulkRequest = bulkRequestSupplier.get();
    // 委托给bulkRequestHandler来执行
    this.bulkRequestHandler.execute(bulkRequest, executionId);
}
```

#### 提交数据

```java
public void execute(BulkRequest bulkRequest, long executionId) {
    Runnable toRelease = () -> {};
    boolean bulkRequestSetupSuccessful = false;
    try {
        // 还记着这哥们吗，提交前的回调
        listener.beforeBulk(executionId, bulkRequest);
        // 允许同时执行的并发度，这里通过信号量来实现
        semaphore.acquire();
        toRelease = semaphore::release;
        // 初始化栅栏，用于等待执行完成
        CountDownLatch latch = new CountDownLatch(1);
        
        // 这里委托给retry进行真正的提交操作
        retry.withBackoff(consumer, bulkRequest, new ActionListener<BulkResponse>() {
            @Override
            public void onResponse(BulkResponse response) {
                try {
                    // 回调
                    listener.afterBulk(executionId, bulkRequest, response);
                } finally {
                    semaphore.release();
                    latch.countDown();
                }
            }

            @Override
            public void onFailure(Exception e) {
                try {
                    // 回调
                    listener.afterBulk(executionId, bulkRequest, e);
                } finally {
                    semaphore.release();
                    latch.countDown();
                }
            }
        });
        bulkRequestSetupSuccessful = true;
        if (concurrentRequests == 0) {
            latch.await();
        }
    } catch (InterruptedException e) {
        // 异常处理
    } finally {
        if (bulkRequestSetupSuccessful == false) {  
            toRelease.run();
        }
    }
}
```

> Retry#RetryHandler#execute

```java
public void execute(BulkRequest bulkRequest) {
    this.currentBulkRequest = bulkRequest;
    // 这里的consumer就是前面构造BulkProcessor传递一个lambda，示例中使用的是BulkAsync
    consumer.accept(bulkRequest, this);
}
```

这里看起来只有短短两行代码，但是却很巧妙，经过层层包装之后，最终其实是使用外部传递进来的处理器，这样子的好处在于，调用方可以根据实际的情况来提交数据，而具体的其余的阈值判断等操作则是由BulkProcessor来执行，函数式编程的一个很巧妙的方式

到这里的话，我们就已经知道了，当达到数量阈值或者大小阈值的时候，触发的提交处理逻辑，其实是采用了延迟触发的方式，只有在添加数据的时候，才去检查是否满足阈值条件。当然，如果单纯通过这种方式来触发提交，就会出现有数据，但是不满足阈值，从而导致要很长的一段时间之内数据才会触发提交甚至不触发提交的情况发生，此时就需要通过另外的阈值--时间，来保障最低频次的提交

#### 定时提交

前面在看BulkProcessor的代码的时候，我们提到了这一行`this.cancellableFlushTask = startFlushTask(flushInterval, scheduler);`，这个就是定时提交的核心了

> startFlushTask

```java
private Scheduler.Cancellable startFlushTask(TimeValue flushInterval, Scheduler scheduler) {
    // 如果flushInterval为空，则表示不设置该任务
    if (flushInterval == null) {
        return new Scheduler.Cancellable() {
            @Override
            public boolean cancel() {
                return false;
            }

            @Override
            public boolean isCancelled() {
                return true;
            }
        };
    }
    
    
    
    final Runnable flushRunnable = scheduler.preserveContext(new Flush());
    // 固定延迟调度，底层还是使用ScheduledThreadPoolExecutor
    return scheduler.scheduleWithFixedDelay(flushRunnable, flushInterval, ThreadPool.Names.GENERIC);
}
```

> Flush

```java
class Flush implements Runnable {

    @Override
    public void run() {
        // 这里锁住了，所以如果定时提交数据的是时候，有人调用add，add会阻塞住
        synchronized (BulkProcessor.this) {
            if (closed) {
                return;
            }
            if (bulkRequest.numberOfActions() == 0) {
                return;
            }
            // 还是回到了execute方法
            execute();
        }
    }
}
```

到了这里，关于BulkProcessor的魔力我们就清楚了，通过设置数量阈值和大小阈值，当add时候会检查是否已经满足提交条件，如果满足，则会触发提交，同时，固定间隔时间之内也会触发提交，从而保障至少在指定的时间间隔之内，数据会被提交一次，当然，两个任务之间可能会同时提交，因此通过`synchronized`来进行同步保障

