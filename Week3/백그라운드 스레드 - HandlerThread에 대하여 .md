- - - # 0️⃣ 오늘의 목표 입니다 : >

      ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/32dd1991-13bf-460c-a52e-ec3d0fab679a/Untitled.png)

      - HandlerThread의 멤버에 대해 이해합니다.
      - HandlerThread가 필요한 이유를 이해합니다.
      - HandlerThread 예시를 이해해봅시다.

      
    
      
    
      # 1️⃣ HandlerThread의 멤버
    
      ## HandlerThread 클래스
    
      ------
    
      - `HandlerThread` 는 `Thread` 클래스를 상속받고, 내부적으로 `looper.prepare()` 와 `looper.loop()` 을 실행하는 Looper 스레드이다.
    
        (Looper 스레드 = Looper를 갖는 스레드)
    
      - `Handler`를 가진 스레드 x, `**Looper` 스레드이면서 `Handler`에서 사용하기 위한 스레드이다.**
    
      - `Handler` 는 `HandlerThread` 에서 생성한 Looper에 연결하는데, 이때 `Handler` 의 세번째 생성자 `Handler(Looper looper, Handler.Callback callback)` 을 사용한다.
    
      ## 필드
    
      ### [`mPriority`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/HandlerThread.java;l=29?hl=ko)
    
      > The priority to run the thread at.
    
      - 스레드르르 실행할 우선 순위
    
      ### [`mTid`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/HandlerThread.java;l=30?hl=ko)
    
      - 현재 스레드의 아이디
      - -1이 디폴트 값
      - `run()` 내부에서 Process.myTid() 로 초기화됨
      - `loop()` 함수 종료 후 -1로 초기화 됨
    
      ### [`mLooper`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/HandlerThread.java;l=31?hl=ko)
    
      - 현재 스레드와 연결된 루퍼
      - HandlerThread의 `run()`에서 자동적으로 연결
    
      ### [`mHandler`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/HandlerThread.java;l=32?hl=ko)
    
      - 현재 스레드의 루퍼와 연결된 핸들러
      - 개발자가 직접 연결
    
      ## 메서드
    
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
    
      - mPriority 값 설정 여부로 생성자가 2개로 나뉜다.
    
        - mPriority 매개변수가 없는 경우엔 0로 초기화 된다.
    
          ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0ba36867-be1c-419b-9e3c-0edd4d234c2a/Untitled.png)
    
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
    
      - `Looper.prepare()`, `Looper.loop()` 호출은 물론
    
      - 멤버변수인 `mLooper`에  `Looper.myLooper()의 반환값` 을 할당하는 일도 한다.
    
      - `getLooper()` 는 바로 `mLooper`를 반환하지 않으며, `quit()` 에서도 `mLooper.quit()` 으로 바로 Looper를 중지시키는 것이 아니라, `getLooper()` 를 통해서 얻은 Looper에 `quit()`을 호출한다.
    
      - 왜 이렇게 할까? HandlerThread는 Looper를 멤버 변수로 갖는다. 만약 Looper가 생성되기 전에
    
        ```
        getLooper()
        ```
    
         의 반환값은 null일 것이고, 이는 NPE로 이어질 수 있다.
    
        ```
        quit()
        ```
    
        에서도 마찬가지로 
    
        ```
        mLooper.quit()
        ```
    
         을 바로 호출한다면, Looper가 아직 생성되지 않았다면 NPE가 발생한다.
    
        - 따라서 `getLooper()` 에서는 mLooper가 할당될 때까지 `Thread` 를 대기시키는 작업을 하고, mLooper가 할당되었을 때 mLooper를 반환하도록 한다.
        - `quit()` 에서도 `getLooper()` 에서 반환된 Looper를 가지고 quit() 을 하도록 하여 NPE가 발생하지 않도록 한다.
    
      ### `getLooper()` 메서드
    
      - ```
        isAlive()
        ```
    
         호출을 통해서 
    
        ```
        HandlerThread
        ```
    
        에서 
    
        ```
        start()
        ```
    
         메서드가 호출되었는지 체크한다. = Thread가 시작되었는지 체크한다.  (다른 스레드도 마찬가지이다.)
    
        - `isAlive()`는 스레드가 `start()`메서드로 시작되었고, 아직 종료되지 않았을 때 `true`를 반환한다. 종료되었을 때는 `false` 를 반환한다.
        - `HandlerThread` 를 사용할 때는 항상 `start()`를 호출하여 스레드가 시작된 후에 사용해야 한다. (특히 `getLooper()` 를 사용할때는 더욱더 유의!)
    
      - HandlerThread가 시작된 상태라면, mLooper가 null 인지 체크한다.
    
        - mLooper가 null이라면 
    
          ```
          wait()
          ```
    
           을 호출해서 mLooper가 할당될 때까지 
    
          ```
          HandlerThread
          ```
    
          를 블락상태로 만든다. → 
    
          ```
          synchronized
          ```
    
          - `wait()`
    
            
    
            - `Object` 클래스에 속한 메서드
            - 외부에서 `wait()`가 호출된 객체의 의 `notify()` 혹은 `notifyAll()` 를 호출할 때까지 thread가 블락상태가 된다.
    
        - 이렇게 하는 
    
          이유?
    
          ```
          HandlerThread
          ```
    
          의 
    
          ```
          start()
          ```
    
           메서드가 호출되고 나서,
    
          ```
          HandlerThread
          ```
    
           의 
    
          ```
          run()
          ```
    
           메서드가 호출되는데, 이 
    
          ```
          run()
          ```
    
           실행되는 시점을 알 수 없다. 따라서 
    
          ```
          getLooper()
          ```
    
          가 실행되는 시점에 mLooper가 null일 수 있다.
    
          - 따라서 mLooper에 Looper가 할당될때까지 대기하기 위해서 while 문을 돌면서 mLooper가 null인지를 계속해서 체크한다.
          - `Thread`에서  `start()` 이 호출되면, JVM에서 해당 `Thread` 의 `run()` 를 호출한다.
          - 호출 결과: 두 스레드(`start()` 를 호출한 스레드, `run()` 를 실행하는 스레드)가 동시에 실행되는 상태가 된다.
    
        - `getLooper()` 가 mLooper가 직접적으로 참조되는 유일한 곳이다. 즉, mLooper를 사용하려면 `getLooper()`를 통해서만 얻을 수 있기에, mLooper로 인한 NPE가 방지된다.
    
      1. 블락 상태인 `Thread`를 깨우는 곳: 에서 `wait()` 를 호출하여 mLooper가 할당될때가지 기다리다가, (2)에서 mLooper 가 할당된 후에 `notifyAll()` 을 통해서 `Thread`를 깨운다.
    
      ### `quit()` 메서드 `quitSafely()` 메서드
    
      여기에 자세히 적어뒀지롱
    
      [메인 스레드와 Handler Part 1 - Handler와 Looper 그리고 MessageQueue의 동작 방식](https://medium.com/write-android/메인-스레드와-handler-part-1-handler와-looper-그리고-messagequeue의-동작-방식-f0bee443d71e)
    
      ### `getThreadHandler` 메서드
    
      - 현재 스레드의 루퍼와 연결된 핸들러가
        - 있다면 → 리턴해준다. mHandler를 리턴한다.
        - 없다면 → 새로 생성해준다. mHandler를 초기화 시켜준다.
    
      ### `getThreadId` 메서드
    
      - 현재 스레드의 아이디를 리턴 해준다.
      - `run()` 함수 호출전까진, 초기값인 -1이 리턴된다.
    
      # 2️⃣ HandlerThread가 필요한 이유
    
      - 메세지 (작업) 에 집중
    
      - 내부적으로 
    
        ```
        looper.prepare()
        ```
    
         와 
    
        ```
        looper.loop()
        ```
    
         을 실행하는 Looper 스레드 → Looper가 없는 상황을 방지
    
        - `Handler(Looper looper, Handler.Callback callback)` 을 사용
    
      - 핸들러가 달려있지 않은 스레드에 대한 메모리릭 방지
    
      # 3️⃣ HandlerThread 예시 → 보완해서 알려주겠다..
    
      ### 순차작업에 HandlerThread를 사용한다.
    
      - **UI 작업을 필요하지 않지만 단일 스레드에서 순차적인 작업을 해야할 때** HandlerThread를 사용하면 된다.
    
      - ex. 아래와 같이 RecyclerView의 각 아이템에 즐겨찾기 버튼이 있을 때, 즐겨찾기 버튼을 누를때마다 즐겨찾기 여부를 실시간으로 DB에 반영한다고 해보자
    
        ![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5ace8f58-ad41-4635-9eca-c17258febdcd/IMG_8193.heic](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5ace8f58-ad41-4635-9eca-c17258febdcd/IMG_8193.heic)
    
        이때 즐겨찾기 버튼을 누를 때마다 새로운 스레드를 생성해서 처리하면, 사용자가 반복해서 즐겨찾기 버튼을 누르는 경우, 스레드는 start()를 호출한 순서대로 실행되지 않기 때문에, 선택 → 해제 → 선택 을 했더라도 선택 → 선택 → 해제 순으로 반영되여 잘못된 결과가 나올 수 있다.
    
        이와 같이 실행 순서를 맞추어야 할때 HandlerThread를 사용한다.
    
      - **HandlerThread를 사용하지 않는 방법: BlockingQueue를 이용하는 방법**
    
        - `BlockingQueue` : 멀티 스레딩에 안전한 Queue 자료구조
    
        - 백그라운드 스레드에서 무한 반복문을 돌면서 BlockingQueue에서 데이터를 가져와서(`take()`) 처리한다.
    
        - 스레드 외부에서는 `BlockingQueue`에 데이터를 넣기(`put()`) 을 실행하면 된다.
    
        - `BlockingQueue` 코드
    
          ```java
          class Producer implements Runnable {
             private final BlockingQueue queue;
             Producer(BlockingQueue q) { queue = q; }
             public void run() {
               try {
                 while (true) { queue.put(produce()); }
               } catch (InterruptedException ex) { ... handle ...}
             }
             Object produce() { ... }
           }
          
           class Consumer implements Runnable {
             private final BlockingQueue queue;
             Consumer(BlockingQueue q) { queue = q; }
             public void run() {
               try {
                 while (true) { consume(queue.take()); }
               } catch (InterruptedException ex) { ... handle ...}
             }
             void consume(Object x) { ... }
           }
          
           class Setup {
             void main() {
               BlockingQueue q = new SomeQueueImplementation();
               Producer p = new Producer(q);
               Consumer c1 = new Consumer(q);
               Consumer c2 = new Consumer(q);
               new Thread(p).start();
               new Thread(c1).start();
               new Thread(c2).start();
             }
           }
          ```
    
      - **하지만 `HandlerThread` 를 사용했을 때가 더 Message에 집중할 수 있다는 장점이 있다.**
    
      - 위의 예제를 HandlerThread를 이용한 구현
    
      ```java
      private Handler favoriteHandler;
      private HandlerThread handlerThread;
      
      @Override
      public void onCreate(Bundle savedInstanceState) {
      	...
      	handlerThread = HandlerThread("Favortie Processing Thread);
      	**handlerThread.start(); // (1)
      	favoriteHandler = Handler(handlerThread.getLooper()) { 
      
      		@Override 
      		public void handleMessage(Message message) { 
      			MessageFavortie messageFavorite = (MessageFavorite) msg.obj;
      			FavoriteDao.updateMessageFavorite(messageFavorite.id, 
      				messageFavorite.favorite)
      		}**	
      	};
      }
      
      private class MessageAdapter extends ArrayAdapter<Message> {
      	@Override
      	public View getView(int position, View containerView, ViewGroup parent) {
      		...
      		holder.favorite.setOnClickListener (new View.OnClickListener() {
      			
      			@Override
      			public void onClick(View view) {
      				boolean checked = ((CheckBox) View).isChecked();
      				Message message = favoriteHandler.obtainMessage();
      				message.obj = new MessageFavorite(item.id, checked);
      				favoriteHandler.sendMessage(message);
      			}
      		}
      	}
      }
      
      @Override
      protected void onDestroy() {
      	**handlerThread.quit(); // (5)**
      	super.onDestroy()
      }
      ```
    
      1. HandlerThread를 시작한다.
      2. HandlerThread의 Looper와 연결된 Handler를 생성한다.
      3. 체크박스를 선택, 해제할 때마다 Handler에 Message를 보낸다.
      4. Message를 받아서 DB에 반영한다.
      5. HandlerThread의 quit() 는 메서드 내부에서 `Looper.quit()` 을 실행해서 Looper를 종료한다.