# 백그라운드 스레드 part1 - HandlerThread 클래스

오늘은  백그라운드 스레드를 구현하는데 많이 사용되는 [HandlerThread](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/HandlerThread.java;l=40?q=handlert&sq=&hl=ko) 이 클래스에 대해 알아보면서, 지난 글에서 다뤄보았던 [Handler와 Looper 그리고 MessageQueue의 동작 방식](https://medium.com/write-android/%EB%A9%94%EC%9D%B8-%EC%8A%A4%EB%A0%88%EB%93%9C%EC%99%80-handler-part-1-handler%EC%99%80-looper-%EA%B7%B8%EB%A6%AC%EA%B3%A0-messagequeue%EC%9D%98-%EB%8F%99%EC%9E%91-%EB%B0%A9%EC%8B%9D-f0bee443d71e) 에 대한 이해를 심화시켜보는 시간을 가져보려고 합니다. HandlerThread의 멤버에 대해 이해하고, HandlerThread가 필요한 이유를 이해해보도록 하겠습니다.

더불어 이 글은 아래 문서들에 대한 자세한 설명입니다. 아래 공식 문서와 FW 코드 전문을 참고하시어 보시길 추천드립니다. 

- [HandlerThread 공식 문서](https://developer.android.com/reference/android/os/HandlerThread)

- [HandlerThread 오픈 소스 전문](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/HandlerThread.java;l=40?q=handlert&sq=&hl=ko) 

## HandlerThread의 멤버

### HandlerThread 클래스

------

<img src="https://user-images.githubusercontent.com/59532818/138552384-4b321918-2698-4713-a147-e8acaa119859.png" alt="image" style="zoom:50%;" />

- `HandlerThread` 는 `Thread` 클래스를 상속받고, 내부적으로 `looper.prepare()` 와 `looper.loop()` 을 실행하는 Looper 스레드이다. (Looper 스레드란 Looper를 갖는 스레드를 의미합니다. )
- `Handler`를 가진 스레드가 아닙니다.  `HandlerThread`는 **`Looper` 스레드이면서 `Handler`에서 사용하기 위한 스레드** 입니다.
- `Handler` 는 `HandlerThread` 에서 생성한 Looper에 연결하는데, 이때 `Handler` 의 세번째 생성자 `Handler(Looper looper, Handler.Callback callback)` 을 사용합니다. 

### 필드

### [`mPriority`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/HandlerThread.java;l=29?hl=ko)

> The priority to run the thread at.

- 스레드를 실행할 우선 순위입니다.

### [`mTid`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/HandlerThread.java;l=30?hl=ko)

- 현재 스레드의 아이디입니다.
  - -1이 디폴트 값으로 설정되어있습니다. 
- `run()` 내부에서 `Process.myTid()` 로 초기화됩니다.
- `loop()` 함수 종료 후 -1로 초기화됩니다. 

### [`mLooper`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/HandlerThread.java;l=31?hl=ko)

- 현재 스레드와 연결된 루퍼입니다. 
- HandlerThread의 `run()`에서 자동적으로 연결됩니다. 

### [`mHandler`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/HandlerThread.java;l=32?hl=ko)

- 현재 스레드의 루퍼와 연결된 핸들러입니다. 
- `mLooper`와는 달리, 개발자가 직접 연결해주어야하는 부분입니다. 

### 메서드

```java
public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    

public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }
```

### `HandlerThread 생성자`

- `mPriority` 값 설정 여부로 생성자가 2개로 나뉩니다. 

  - `mPriority` 매개변수가 없는 경우엔 0로 초기화 됩니다. 

    <img src="https://user-images.githubusercontent.com/59532818/138552387-9f5f16eb-80ea-4337-a974-afea592bfd82.png" alt="image" style="zoom: 50%;" />

```java
public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
	public Looper getLooper() {
		**if(!isAlive()) {** 
			return null;
		}

		synchronized(this) {
			**while(isAlive() && mLooper == null) {** 
				try { 
					**wait(); // (5)**
				} catch (InterruptedException e) {
				}
			}
		}
		return mLooper;
	}
	public boolean quit() {
		Looper looper = getLooper();
		if (looper != null ){
			looper.quit()
			return true
		}
		return false
	}

public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
 }
@NonNull
    public Handler getThreadHandler() {
        if (mHandler == null) {
            mHandler = new Handler(getLooper());
        }
        return mHandler;
    }
public int getThreadId() {
        return mTid;
    }
```

### `run()`메서드

- `Looper.prepare()`, `Looper.loop()` 호출은 물론 멤버변수인 `mLooper`에  `Looper.myLooper()의 반환값` 을 할당하는 일도 합니다. 

- `getLooper()` 는 바로 `mLooper`를 반환하지 않으며, `quit()` 에서도 `mLooper.quit()` 으로 바로 Looper를 중지시키는 것이 아니라, `getLooper()` 를 통해서 얻은 Looper에 `quit()`을 호출합니다.

- 왜 이렇게 할까? `HandlerThread`는 Looper를 멤버 변수로 갖는다. 만약 Looper가 생성되기 전에 `getLooper()` 의 반환값은 null일 것이고, 이는 NPE로 이어질 수 있습니다. `quit()`에서도 마찬가지로 `mLooper.quit()` 을 바로 호출한다면, Looper가 아직 생성되지 않았다면 NPE가 발생합니다.

  - 따라서 `getLooper()` 에서는 `mLooper`가 할당될 때까지 스레드를 대기시키는 작업을 하고, `mLooper`가 할당되었을 때 `mLooper`를 반환하도록 합니다.
  - `quit()` 에서도 `getLooper()` 에서 반환된 Looper를 가지고 `quit()` 을 하도록 하여 NPE가 발생하지 않도록 합니다.

### `getLooper()` 메서드

- `isAlive()` 호출을 통해서 `HandlerThread`에서 `start()` 메서드가 호출되었는지 체크합니다. 즉, Thread가 시작되었는지 체크합니다.  
  
  - `isAlive()`는 스레드가 `start()`메서드로 시작되었고, 아직 종료되지 않았을 때 `true`를 반환합니다. 종료되었을 때는 `false` 를 반환합니다.
  - `HandlerThread` 를 사용할 때는 항상 `start()`를 호출하여 스레드가 시작된 후에 사용해야 합니다. (특히 `getLooper()` 를 사용할때는 더욱더 유의!)
  
- `HandlerThread`가 시작된 상태라면, `mLooper`가 null 인지 체크합니다.

  - `mLooper`가 null이라면 `wait()` 을 호출해서 `mLooper`가 할당될 때까지 `HandlerThread`를 블락상태로 만듭니다. 이 블락 상태는 

    `synchronized` 구문으로 이뤄집니다. 

    - `wait()` 메서드는 `Object` 클래스에 속한 메서드입니다. 
      - 외부에서 `wait()`가 호출된 객체의 의 `notify()` 혹은 `notifyAll()` 를 호출할 때까지 thread가 블락상태가 됩니다.

  - 이렇게 하는 이유?

    `HandlerThread`의 `start()` 메서드가 호출되고 나서, `HandlerThread` 의 `run()` 메서드가 호출되는데, 이 `run()` 실행되는 시점을 알 수 없습니다. 따라서 `getLooper()`가 실행되는 시점에 mLooper가 null일 수 있습니다.

    - 따라서 `mLooper`에 Looper가 할당될때까지 대기하기 위해서 while 문을 돌면서 `mLooper`가 null인지를 계속해서 체크합니다.
    - `Thread`에서  `start()` 이 호출되면, JVM에서 해당 `Thread` 의 `run()` 를 호출합니다.
    - 호출 결과: 두 스레드(`start()` 를 호출한 스레드, `run()` 를 실행하는 스레드)가 동시에 실행되는 상태가 됩니다.

  - `getLooper()` 가 `mLooper`가 직접적으로 참조되는 유일한 곳이다. 즉, `mLooper`를 사용하려면 `getLooper()`를 통해서만 얻을 수 있기에, `mLooper`로 인한 NPE가 방지됩니다.

1. 블락 상태인 `Thread`를 깨우는 곳: 에서 `wait()` 를 호출하여 `mLooper`가 할당될때가지 기다리다가, (2)에서 `mLooper` 가 할당된 후에 `notifyAll()` 을 통해서 `Thread`를 깨웁니다.

### `quit()` 메서드 `quitSafely()` 메서드

해당 메서드들에 대한 설명은 [직전 글의 *Looper의 동작* 부분](https://medium.com/write-android/메인-스레드와-handler-part-1-handler와-looper-그리고-messagequeue의-동작-방식-f0bee443d71e) 에서 자세히 설명하고 있으니 참고 부탁드립니다. 

### `getThreadHandler` 메서드

- 현재 스레드의 루퍼와 연결된 핸들러가
  - 있다면 → 리턴해준다. `mHandler`를 리턴합니다.
  - 없다면 → 새로 생성해준다. `mHandler`를 초기화 시켜줍니다.

### `getThreadId` 메서드

- 현재 스레드의 아이디를 리턴합니다.
- `run()` 함수 호출전까진, 초기값인 -1이 리턴합니다.

## HandlerThread가 필요한 이유

- 내부적으로 *looper.prepare()*와 *looper.loop()* 을 실행하는 Looper 스레드에게 Looper가 없는 상황을 방지해줍니다. 

  - 해당 루퍼와 연결될 핸들러를 생성할 때엔 항상 `Handler(Looper looper, Handler.Callback callback)` 생성자를 사용하여, 루퍼를 명시적으로 정해줍니다. 
  
- 핸들러가 달려있지 않은 스레드에 대한 메모리릭을 방지해줍니다. 

- 개발자가 신경써야하는 위와 같은 상황들을 FW단에서 처리할 수 있게 하여, 개발자가 메세지 (작업) 에 집중할 수 있게 해줍니다. 
