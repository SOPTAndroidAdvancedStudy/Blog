

# 메인 스레드와 Handler - Handler와 Looper 그리고 MessageQueue의 동작 방식 

안드로이드의 메인스레드인 ActivityThread.java 를 이해하는데에 큰 배경 지식이 된다는 점에서 Handler와 Looper 그리고 MessageQueue의 동작 방식을 이해하는 것은 큰 가치가 있는 일입니다. 스레드 통신과 약간의 자료구조에 대한 부분까지 생각해보는 좋은 기회가 될 것입니다. 따라서 이번 아티클에서는 안드로이드 프레임워크의 코드와 함께 Handler와 Looper 그리고 MessageQueue의 동작 방식에 대해 깊게 알아보겠습니다. 



## 안드로이드의 메인스레드

안드로이드의 메인스레드는 어디에 있을까요 ? 프로그램의 시작인 main 함수를 따라가다보면, 안드로이드 프레임워크의 내부 클래스인 [ActivityThread.java](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/ActivityThread.java;l=1?q=ActivityThread.java&sq=&hl=ko)에 도달하게 됩니다. 즉, ActivityThread.java의 main 함수가 안드로이드 애플리케이션의 시작점이자 메인스레드가 되는 곳입니다. 이번 글에서 다루고자 하는 Handler-Looper 구조 중심으로 코드를 보면, 이들이 어떤 흐름으로 사용되는지 짐작할 수 있습니다. 

```java
/**
 * This manages the execution of the main thread in an
 * application process, scheduling and executing activities,
 * broadcasts, and other operations on it as the activity
 * manager requests.
 *
 * {@hide}
 */
public final class ActivityThread extends ClientTransactionHandler {
    /** @hide */
    public static void main(String[] args) {
        // 생략

        Looper.prepareMainLooper();

        // 생략
      
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
}
```

마지막에 호출된 Looper.loop()가 끝나면 정상적인 함수 종료가 아닌, RuntimeException을 터뜨리는데, 과연 loop()함수는 어떤 일을 하는 함수이기에 RuntimeException을 터뜨리는 걸까요 ? 

## Looper 

안드로이드의 메인스레드의 시작점인 ActivityThread.java에서 Handler-Looper 구조를 확인하였습니다. Looper와 loop() 함수에 대해 더 알아보겠습니다. 

### Looper의 역할

Looper라는 이름에서도 알 수 있듯이 하나의 반복 작업을 맡고 있는 역할로 이해할 수 있습니다. 실제로 프로세스가 종료될 때 까지, 무한 반복문을 통해 반복 작업을 수행하게 됩니다. 여기서 Looper가 맡는 반복작업은 구체적으로 MesseageQueue에 있는 message, runnable을 계속해서 하나씩 꺼내주는 작업이 되겠습니다. 

### Looper의 생성

그렇다면 Looper는 어떻게 생성되고 관리될까요 ? 일단 기본적으로 Looper는 각 스레드에 종속되며, MesseageQueue는 각 Looper에 종속됩니다. 

먼저 각 Looper는 스레드 상관없이, 자신을 생성한 스레드의 [Thread Local Storage (TLS)](https://docs.microsoft.com/ko-kr/windows/win32/procthread/thread-local-storage?redirectedfrom=MSDN)에 **저장**됩니다. 하지만 스레드가 메인 스레드이냐 아니냐에 따라 Looper의 **생성**은 다른 방식으로 이뤄집니다. 

#### 메인 스레드의 Looper 

메인 스레드의 Looper는 안드로이드 환경에서 직접 생성해줍니다. 

**prepareMainLooper()** 가 deprecated된 이후(API 30 이후), **안드로이드 환경에서 MainLooper를 직접 생성**해줍니다. 따라서 개발자들은 MainLooper의 생성에 관여하지 않고, [getMainLooper()](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Looper.java;drc=master;bpv=1;bpt=1;l=135?hl=ko&gsn=getMainLooper&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Djava%3Fpath%3Dandroid.os.Looper%2361aaa387207f8cedf23a7e053a4ec96faa270a8d5f295bcbec5f3acb7f17df25)를 통해 가져다 쓰기만 하면 됩니다.

#### 일반 스레드의 Looper

일반 스레드의 Looper는 개발자가 생성합니다. 

**일반스레드**의 Looper를 생성하는 방법은 **prepare() 함수를 사용**하는 것 입니다. 또한 메인스레드와 마찬가지로, loop() 호출을 통해 동작하게 됩니다. 정리하면, 개발자는 일반스레드를 사용하는 곳에서 **prepare() 호출**을 통해 **Looper**를 **생성**하고 **loop() 호출**에 의해 **Looper**를 **동작**하게 합니다.

### Looper의 동작

Looper의 동작을 이해하기 위해 Looper.loop() 메서드의 주요 코드를 확인해보겠습니다. loop() 함수안은 *pop한 내용물이 null일때*까지 반복하는 무한 loop로 구성되어있습니다. 이 무한 loop안에서, *msg.target.dispatchMessage(msg);* 를 통해, Handler인 *target*이 작업을 처리하게 만들어줍니다. 

그렇다면 과연 *pop한 내용물이 null일때* , 즉 if (msg == null) 가 언제 참이 될까요 ? 바로 Looper를 종료하는 메서드인 quit(), quitSafely()가 호출될 때 입니다. 두 메서드는 결국 MesseageQueue의 quit()를 호출하며, 이의 결과로 queue.next()에서 null 반환되어 위의 if문을 통과하게 되는 것입니다. 이로써 Looper가 종료됩니다. 

```java
/**
 * Run the message queue in this thread. Be sure to call
 * {@link #quit()} to end the loop.
 */ 
public static void loop() {
        final Looper me = myLooper();        
   // 생략
        final MessageQueue queue = me.mQueue;
   // 생략
        for (;;) { // 무한 루프 시작 ‼️
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
          	// 생략
            try {
                msg.target.dispatchMessage(msg);
                if (observer != null) {
                    observer.messageDispatched(token, msg);
                }
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } catch (Exception exception) {
                if (observer != null) {
                    observer.dispatchingThrewException(token, msg, exception);
                }
                throw exception;
            } finally {
                ThreadLocalWorkSource.restore(origWorkSource);
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
   					// 생략
            msg.recycleUnchecked();
        } // 무한 루프 끝 ‼️
    }
```

## Message와 MessageQueue

### Message와 MessageQueue의 역할 

Message란 실제 수행해야할 작업에 대한 명세가 들어있는 자료구조이며, MessageQueue는 Message를 담는 자료구조 입니다. 이를 Looper와 관련지어 본다면, MessageQueue는 Looper의 반복 대상이며, Message는 Looper의 실질적인 반복의 결과물로 이해할 수 있습니다.

### Message 생성

Message를 생성할 때는 사용한 Message 객체를 다시 초기화하여 재사용하는 방식인 오프젝트 풀 방식을 사용합니다. 따라서 **Message 객체는 Message.obtain() 이나 Handler.obtainMessage()을 통해 얻고 관리되어야합니다**.

### MessageQueue 동작

#### Message 클래스와 타임스탬프 

MessageQueue는 Message를 담고 있는 자료구조라 말씀드렸습니다. 과연 Message는 어떤 방식으로 MessageQueue에 쌓이고, MessageQueue는 어떻게 이 Message를 처리하는지 알아보겠습니다. 

먼저 [Message 클래스](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Message.java?q=public%20final%20class%20Message%20implement%20Parcelable&ss=android&hl=ko) 에는 핸들러가 스레드간의 통신 매개로서 전달해야할, (다른 스레드에서 실행될) 작업에 대한 정보를 담고있습니다. 

```
public final class Message implements Parcelable {
    /**
     * User-defined message code so that the recipient can identify
     * what this message is about. Each {@link Handler} has its own name-space
     * for message codes, so you do not need to worry about yours conflicting
     * with other handlers.
     */
    public int what;

    /**
     * arg1 and arg2 are lower-cost alternatives to using
     * {@link #setData(Bundle) setData()} if you only need to store a
     * few integer values.
     */
    public int arg1;


  // 생략 !!
     
    /**
     * The targeted delivery time of this message. The time-base is
     * {@link SystemClock#uptimeMillis}.
     * @hide Only for use within the tests.
     */
    @UnsupportedAppUsage
    @VisibleForTesting(visibility = VisibleForTesting.Visibility.PACKAGE)
    public long when;

    /*package*/ Bundle data;

    @UnsupportedAppUsage
    /*package*/ Handler target;

    @UnsupportedAppUsage
    /*package*/ Runnable callback;

    // sometimes we store linked lists of these things
    @UnsupportedAppUsage
    /*package*/ Message next;
```



#### MessageQueue가 Linked 구조로 구현된 이유 with 타임 스탬프 

MessageQueue에 Message가 enqueue되는 알고리즘에는 타임스탬프(when + delay) 라는 변수 영향을 줍니다. Message를 enqueue하는 데에는 when이라는 값 이외에 개발자가 delay라는 값을 설정할 수 있습니다. 즉 타임스탬프 값에 따라, MessageQueue의 중간에 enqueue해야하는 경우가 발생하게 빈번히 발생하게 됩니다. 중간 삽입에 편리한 구조를 사용한다는점에서, 배열 구조가 아닌 링크 구조로 구현되어있다고 이해할 수 있습니다. 이 밖에, 링크 구조는 일반적으로 요소의 개수 제한이 없어, 수많은 Message를 담아야하는 MessageQueue에게 적합하다고 볼 수 있습니다. 한마디로, MessageQueue는 Message를 담는 자료구조이며, 개수 제한과 중간 삽입 속도이 쉬운 링크구조의 Queue로 구현되어있다고 요약하겠습니다. 



## Handler

### Handler 역할 두가지 

Handler가 가진 역할을 크게 아래 두가지로 요약할 수 있습니다. 

- Message를 MessageQueue에 보내는 역할
- MessageQueue에서 꺼낸 Message를 처리하는 역할 (MessageQueue에서 Message를 꺼내는 역할은 Looper의 역할입니다.)

### Handler 생성

Handler가 작업을 처리하기 이전에, Handler는 (MessageQueue에서 Message를 꺼내줄) Looper와 연결되어있어야합니다. 따라서 Handler의 생성 또한 Looper와 연결지어 이해해야합니다. 이는 공식문서에서도 강조하는 내용이기도 하는데요, 생성자 부분을 확인해보겠습니다.

[공식문서](https://developer.android.com/reference/android/os/Handler)를 통해 Looper를 명시적으로 지정해주지 않는 Handler 생성자는 모두 deprecated된 것을 확인 하실 수 있습니다. 

<img src="https://user-images.githubusercontent.com/59532818/133730047-af070c93-e771-4c1b-a737-227a3d6269e4.png" alt="image" style="zoom:67%;" />

이는 암시적으로 Looper를 선택했을 때, 아래와 같은 문제가 발생할 수 있기 때문입니다.

- Handler가 새 작업을 기대하지 않고 종료하는 경우의 작업이 자동으로 손실되는 bugs 
- Looper가 활성화되지 않은 스레드에서 Handler가 생성되는 경우의 crashes
- Handler가 연결된 스레드가 개발자가 예상한 것과 다를 때의 Race conditions

> *Implicitly choosing a Looper during Handler construction can lead to bugs where operations are silently lost (if the Handler is not expecting new tasks and quits), crashes (if a handler is sometimes created on a thread without a Looper active), or race conditions, where the thread a handler is associated with is not what the author anticipated.* 

### Handler 동작

앞서 말씀드렸던 Handler의 두 가지 역할 중 Message를 MessageQueue에 보내는 역할이 실질적으로는 어떻게 동작되는지, 관련 메서드들을 살펴보겠습니다. Handler는 MessageQueue에 Message 보낼 때, **Message의 형태**와 **Message의 처리 시점**에 따라 다양한 메서드를 사용합니다. 

먼저 Message의 형태에 따라서 send 방식과 post 방식으로 나뉘게 됩니다.

- send로 시작하는 메서드는 Message를 보냅니다.
- post로 시작하는 메서드는 Runnable형태의 Message를 다룹니다.

또한 Message의 처리 시점에 따라 아래와 같은 함수들을 호출할 수 있습니다.

- ~Delayed()
- ~AtTime()
- ~AtFrontOfQueue()

## 요약

- Handler와 Looper 그리고 MessageQueue가 어떻게 동작하는지 안드로이드 프레임 워크를 통해 알아보았습니다.
- 안드로이드의 메인스레드는 ActivityThread.java의 main() 입니다.
- Looper는 MesseageQueue에 있는 message, runnable을 계속해서 하나씩 꺼내주는 역할을 맡고 있습니다.
- Message란 실제 수행해야할 작업에 대한 명세가 들어있는 자료구조이며, MessageQueue는 Message를 담는 자료구조 입니다. 
- Handler는 Message를 MessageQueue에 보내는 역할과 MessageQueue에서 꺼낸 Message를 처리하는 역할을 맡고 있습니다.