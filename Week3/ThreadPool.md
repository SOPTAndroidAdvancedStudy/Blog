

# TheadPool 이란?

작업 처리에 사용되는 Thread를 미리 제한된 개수만큼 정해 놓고 작업 큐에 들어오는 작업을 스레드에 할당해 처리하는 것.

예를 들어 A건물 1층에서 선물이 포장되고 이 선물이 나오면 B건물 1층에 옮겨줄 작업을 해야하는 고용주가 된 입장이라면?

선물이 나와서 옮기는 요청이 있을때마다 작업자를 고용해도 옮기는 일은 할 수 있지만

고용하고 옮기기에 퍼포먼스가 상당히 나쁘다.

따라서 미리 몇명의 작업자를 고용하고 요청이 있을때마다 그저 옮기도록 작업자에게 배치하면 옮기는 작업에 있어 퍼포먼스를 향상할 수 있다.



Thread도 동일하게 요청마다 Thread생성하고 종료하고를 하면 퍼포먼스에 영향이 간다.

따라서 대기상태 스레드를 미리 준비하고 이를 사용해 스레드 생성/종료 오버해드를 줄여 퍼포먼스 향상에 도움을 줄 수 있다.



## ThreadPool의 장단점

### ThreadPool의 장점

➡ **프로그램 성능 저하를 방지할 수 있다.**

- 스레드 생성/ 종료 오버헤드를 줄임으로써, 많은 비동기 작업시 퍼포먼스를 향상시킨다.

- 스레드 풀을 미리 생성하기에 초기 생성 비용은 들어가지만 이전 스레드를 재사용할 수 있어 시스템 자원을 줄일 수 있다.

- 작업 요청시 스레드가 대기중 상태에서 작업을 실행하기에 딜레이가 발생하지 않는다

### TheadPool의 단점

너무 많은 thread를 생성하고 사용하지 않으면 메모리 낭비가 발생한다.

- 100개가 만들어졌으나 10개만 사용된다면 90개의 스레드는 메모리만 차지한다.

# ThreadPool 의 작동방법

![img](https://upload.wikimedia.org/wikipedia/commons/thumb/0/0c/Thread_pool.svg/400px-Thread_pool.svg.png)

앱에서 사용자로부터 들어오는 요청을 큐에 넣으면

스레드풀에서는 미리 생성한 스레드 큐에 작업을 할당한다.

작업을 처리하면 결과값을 반환한다.

# Java에서 ThreadPool의 구현, ThreadPoolExecutor

java에서는 ThreadPool을 쉽게 생성하고 사용하도록` java.util.concurrent` 에서 `ExecutorService` Interface와 `Executors` class를 제공한다.

`Executors` 의 static method들을 이용하여 `ExecutorService` 구현 객체를 만들어서 ThreadPool을 사용할 수 있다.

자주 사용되는 아래와 같은 함수들이 있다. 

아래 함수들로 하여 Java에서 ThreadPool을 구현한 `ThreadPoolExecutor`를 얻을 수 있다.

```java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }

    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

**위 함수들로 하여 ThreadPool을 생성하고, ThreadPool이 구현된 ThreadPoolExecutor를 얻을 수 있다.**



ThreadPoolExecutor 를 살펴보면 4가지 생성자가 있다.

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)
```

내부적으로 들어온 값에 Validate가 진행되며 받은 인자를 맴버 변수에 저장한다.

ThreadPoolExecutor 변수 주석을 살펴보면

- `corePoolSize` :  ``allowCoreThreadTimeOut()` 로 설정되지 않았다면 유휴 상태 스레드여도 스레드 풀에서 유지할 스래드의 수
- `maximumPoolSize` : 스레드풀에서 허용하는 최대 스레드 수
- `keepAliveTime` : `corePoolSize` 보다 큰 수의 스레드가 있는경우, 유휴상태의 스레드가 종료하기 전 새 작업을 기다릴 수 있는 최대의 시간.
- `unit` : `keepAliveTime` 의 시간 단위를 나타낸다.
- `workQueue` : Task를 실행하기전 대기시키는 큐, execute method에서 제출된 Runnable 작업들이 보관되는 곳
- `threadFactory` : Executor에서 Thread를 만들때 사용되는 팩토리
- `handler` : 실행이 blocked 되었을때 사용할 handler, Thread가 Bound되거나 큐 용량이 꽉 찬경우 사용된다.



위 변수들을 살펴보면 `corePoolSize` 는 스레드 풀에서 유지할 스레드의 수 이고 `maximumPoolSize` 는 허용하는 최대 스레드 수라는 점에서 오해가 생길 수 있다.

`corePoolSize` 만큼 미리 스레드를 만들어 두는 것인가 하는 오해가 생길 수 있으나. 

말그대로 **`corePoolSize`는 스레드 풀에서 유지할 스레드 수이다. 미리 만들어두는 스레드 수가 아니다.**

이러한 부분을 살펴보기 위해 `execute()` 함수를 살펴보면 아래와 같다.

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```



```java
        /*
		 * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         */
```

1. 실행중인 Thread가 `corePoolSize` 보다 적은 경우  새로운 Thread를 시작하고 들어온 Runnable(command) 작업을 처리하고 return 하게된다.

``` java
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
        }
```

해당 부분 코드를 보면 설명과 달리 새로운 스레드를 시작하거나 runnable을 처리하는 부분이 아닌 `addWorker()`를 볼 수 있다.

즉 1 에서 설명된 부분은 `addWorker()` 에서 된다는 것을 알 수 있다.

 `addWorker()`를 따라가면 수많은 내용이 있지만 스레드 생성과 할당 부분을 제외한 부분을 생략하면 다음과 같다.

```java
private boolean addWorker(Runnable firstTask, boolean core) {
        // 상태 체크 부분

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask); // [1]
            final Thread t = w.thread;
            if (t != null) {
                // ...
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int c = ctl.get();

                    if (isRunning(c) ||
                        (runStateLessThan(c, STOP) && firstTask == null)) {
                        // ...
                        workers.add(w); // [2]
                        workerAdded = true;
                        // ...
                    }
                } finally {
                    // ...
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```



[1] 부분에서 Worker 객체를 만들고 task를 할당하는 것을 볼 수 있다. 

```java
w = new Worker(firstTask); // [1]
final Thread t = w.thread;
```

바로 아래에선 Worker.thread를 참조하는 변수를 만들었는데, ThreadPoolExecutor 에서 스레드에 Task를 직접 할당하는 방법이 아닌 Worker를 통해 할당된다.

[2] 에서는 생성한 worker를 wokers 라는 곳에 추가한다.

workers는 스레드풀에 있는 모든 Wocker를 포함합니다.



```java
        /*
		 * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         */
```

2. 작업이 성종적으로 큐에 포함될 수 있는 경우, 스레드 추가를 했어야하는지 or 해당 메소드 진입후 스레드풀이 종료되었는지 등을 더블체크한다.

1번에서 생성된 스레드가 `corePoolSize` 보다 적은 경우는 바로 Worker를 통해 Thread를 만들고 관리했지만

이미 `corePoolSize`의 Thread가 있다면 2번으로 넘어간다.

```java
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
```

`workQueue.offer(command)` 로 큐에 task를 추가한다.

만약 이 과정에서 스레드풀이 shotdown 된 경우 `reject(command)`를 호출하며, 다시 체크했을 때 스레드가 없었다면 스레드를 만든다.

`workQueue.offer(command)` 로 큐에 task가 추가되었다면 Worker에서 `runWoker()` 에서 큐에서 해당 task를 꺼내 쓰는 것을 확인 할 수 있다.

```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                // ...
                try {
                    // ...
                    try {
                        task.run();
                        // ...
                    } catch (Throwable ex) {
                        // ...
                    }
                } finally {
                    // ...
                }
            }
            // ...
        } finally {
            // ...
        }
    }
```

해당 메소드 while문에서 runnable 인터페이를 상속한 task를 run하고있다.

해당 task를 getTask()를 통해 가져온다.

```java
private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            // ...
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            
            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```





```java
        /*
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
```

3. 만약 1,2번 과정을 거치고 더이상 큐에 넣을 수 없다면, 새로운 스레드를 추가한다.
   이때 스레드 생성에 실패하는 경우`reject(command)`를 호출한다.

```java
        else if (!addWorker(command, false))
            reject(command);
```



여기서 아까 넘어온 addWorker부분의 검증 부분을 살펴보면 다음과 같다.

```java
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (int c = ctl.get();;) {
            // Check if queue empty only if necessary.
            // ...
            for (;;) {
                if (workerCountOf(c)
                    >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
                    return false;
                //...
            }
        }
        // ...
    }
```

`addWoker(firstTask, core)` 에서 core에 전달되는 boolean 값으로 추가할 수 있는지 확인하여 반환한다.

core가 true로 들어오는 1번 경우는

worker의 수가 `corePoolSize` 보다 적은 경우는 if문 내부를 무시하고 진행된다.

core가 false로 들어오는 2,3번 경우는

worker의 수가 `maximumPoolSize` 보다 적은 경우는 if문 내부를 무시하고 진행되지만,

큰경우는 더이상 추가할 수 없으므로 false를 return하여

`!addWorker(command, false)` 해당 조건에 걸려 `reject(command)`를 호출한다.



