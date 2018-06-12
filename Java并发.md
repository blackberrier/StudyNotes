## Guava并发：ListenableFuture
### 基本原理
ListenableFuture是可以监听的Future，它是对java原生Future的扩展增强。  
Future表示一个异步计算任务，当任务完成时可以得到计算结果。如果我们希望一旦计算完成就拿到结果展示给用户或者做另外的计算，就必须使用另一个线程不断的查询计算状态。这样代码复杂而且效率低下。  
使用ListenableFuture Guava帮我们检测Future是否完成了，如果完成就自动调用回调函数，这样可以减少并发程序的复杂度。
ListenableFuture是一个接口，它从jdk的Future接口继承，添加了void addListener(Runnable listener, Executor executor)方法。  
###使用
1. 定义ListenableFuture的实例
首先通过MoreExecutors类的静态方法listeningDecorator方法初始化一个ListeningExecutorService的方法，然后使用此实例的submit方法即可初始化ListenableFuture对象。
  ```
  ListeningExecutorService executorService = MoreExecutors.listeningDecorator(Executors.newCachedThreadPool());
  final ListenableFuture<Integer> listenableFuture = executorService.submit(new Callable<Integer>() {
    @Override
    public Integer call() throws Exception {
      System.out.println("call execute..");
      TimeUnit.SECONDS.sleep(1);
      return 7;
    }
  });
  ```
我们上文中定义的ListenableFuture要做的工作，在Callable接口的实现类中定义，这里只是休眠了1秒钟然后返回一个数字7。
2. 有了ListenableFuture实例，执行此Future并执行Future完成之后的回调函数。  
有两种方法
方法一：通过ListenableFuture的addListener方法
```
        listenableFuture.addListener(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println("get listenable future's result " + listenableFuture.get());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (ExecutionException e) {
                    e.printStackTrace();
                }
            }
        }, executorService);
```

方法二：通过Futures的静态方法addCallback给ListenableFuture添加回调函数
```
        Futures.addCallback(listenableFuture, new FutureCallback<Integer>() {
            @Override
            public void onSuccess(Integer result) {
                System.out.println("get listenable future's result with callback " + result);
            }

            @Override
            public void onFailure(Throwable t) {
                t.printStackTrace();
            }
        });
```
