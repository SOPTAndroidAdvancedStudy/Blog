# AsyncTask

백그라운드 스레드에서 테스크를 수행하는 방식은 여러 가지가 있었다. AsyncTask의 B(born)와 D(deprecated)를 알아보자.

## B: Born

안드로이드에서는 메인 스레드를 사용하여 UI 작업을 수행한다. 네트워크 통신을 하거나, Room을 이용하여 데이터를 가져오는 경우 등 다양한 경우에 백그라운드 스레드에서 작업을 수행해야 하는 상황이 발생한다. 그렇다면 백그라운드 스레드에서 작업한 결과를 어떻게 UI에 반영할 수 있을까?



### Handler를 이용한 방법

Handler와 Message 객체를 이용하여 하는 방법을 떠올릴 수 있을 것이다.

- 백그라운드 스레드에서 Handler의 sendMessage() 메서드를 이용하여 Handler로 message객체를 전달한다.

- Handler의 handleMessage(Message msg) 메서드에서 UI 작업을 수행할 수 있다.

  ```java
  private final Handler myHandler = new Handler(getMainLooper()) {
      @Override
      public void handleMessage(@NonNull Message msg) {
          // UI 작업
      }
  };
  
  private void onClick(View view) {
      new Thread(new Runnable() {
   
          @Override
          public void run() {
              // Message 생성
              myHandler.sendMessage(message);
          }
      }).start();
  }
  ```


- 이 방법을 사용하기 위해선 Handler와 Looper에 대해 학습하고, 정확한 사용법을 배우는 것이 필수적이다.
  - Background 작업을 위해 Handler, Thread 클래스를 중복해서 만들어야 한다. (BoilerPlate 코드 발생)
- 그래서 Android FrameWork에서 이 작업을 보다 간편하게 해줄 클래스인 AsynTask를 사용하여 Background작업을 할 수 있도록 하였다.



### AsyncTask를 이용한 방법

**AsyncTask was intended to enable proper and easy use of the UI thread.**

- 공식문서에 나와 있는 것과 같이  AsyncTask는 UI 스레드를 쉽게 사용하기 위한 탄생했다.

- AsyncTask는 클래스 내부에 Handler를 만들어 놓고 특정 메서드들을 구현하기만 하면 백그라운드 스레드에서 작업 결과를 UI에 반영할 수 있다. 메서드들은 다음과 같다.

  ```java
  @MainThread @Deprecated
  protected void onPreExecute() {
  }
  
  @WorkerThread @Deprecated
  protected abstract Result doInBackground(Params... params);
  
  @MainThread @Deprecated
  protected void onProgressUpdate(Progress... values) {
  }
  
  @MainThread @Deprecated
  protected void onPostExecute(Result result) {
  }
  ```

- doInBackground() 메서드를 제외한 나머지는 전부 *@MainThread annotation*이 붙어 있는 형태이다. 다시 말해서 **UI 작업**이 이루어질 수 있는 곳이다.

- `onPreExecuted()`: **작업을 시작하기 전에 무엇을 할 것인가?**

  - 작업이 실행되기 전에 UI 스레드에서 호출

- `doInBackground()`: **백그라운드 스레드에서 어떤 작업을 수행할 것인가?**

  - onPreExecuted() 직후 백그라운드 스레드에서 호출
  - publishProgress(Progress...) 메서드를 통해 값을 publish

- `onProgressUpdate(Progress... values)`: **백그라운드 스레드에서 전달 받은 결과를 가지고 무엇을 할 것인가?**

  - publishProgress(Progress...) 호출 이후에 UI 스레드에서 호출

- `onPostExecute(Result result)`: **백그라운드 스레드 작업이 전부 끝난 이후에 무엇을 할 것인가?**

  - 백그라운드 작업의 결과가 매개변수로 전달
  - 백그라운드 스레드 작업이 전부 끝난다음에 UI 스레드에서 호출



#### AsyncTask Example

아래의 예시 코드는 Background Thread에서 ***MAX*** 만큼 반복하면서 반복한 값을 UI Thread로 보내주는 코드이다.

```java
public class MyAsyncTask extends AsyncTask<Void , Integer , Boolean> {

    TextView textView;
    
    private int MAX = 100000000;

    public MyAsyncTask(TextView textView){
        this.textView = textView;
    }

    @Override
    protected Boolean doInBackground(Void... voids) {
        for (int i = 0; i < MAX ; i++){
            publishProgress(i);
        }
        return true;
    }

    @Override
    protected void onPreExecute() {
        super.onPreExecute();
    }

    @Override
    protected void onPostExecute(Boolean s) {
        textView.setText("AsyncTask2 End");
        super.onPostExecute(s);
    }

    @Override
    protected void onProgressUpdate(Integer... values) {
        textView.setText(values[0].toString());
        super.onProgressUpdate(values);
    }
}
```

> AsyncTask 예제 1

### D: deprecated

그렇다면 AsyncTask는 왜 Deprecated 되었을까? AsyncTask는 보다 편리하게 backgroundThread 작업을 만들어 줬지만 처리해 줘야 하는 부분이 많다. 결론부터 얘기하자면 AsyncTask는 문제가 있는 API여서 Deprecate 된 게 아닌 소프트웨어 기술이 발달했기 때문에 자연스레 사라진 것이라고 생각한다. 믹서기가 생긴 이후로 맷돌은 거의 모습을 감추었다. *여기서 맷돌이 AsyncTask다.*

1. 메모리 문제 발생 가능

   - 메모리 누수가 굉장히 흔하게 일어난다. Activity와 종료 시점이 다르기 때문에 Activity가 종료된 이후에도 AsyncTask는 계속 자기 할 일을 한다.
   - *AsyncTask 예제 1* 의 코드에서 진행 중인 Activity를 종료한다면 어떻게 될까? Activity는 죽었지만 위의 MyAsyncTask의 작업은 계속 진행된다. 이 상태가 지속되면 이는 즉 메모리 누수를 의미한다. (그림 1은 실제로 LeakCanary를 사용해 메모리 누수를 탐지해본 사진)
   - ![fdsa](C:\Users\thdgn\OneDrive\바탕 화면\asyncTask\fdsa.PNG)
                                              그림 1

2. 순차 실행으로 인한 속도 저하

   - AsyncTask가 생성되는 곳은 Ui Thread에서 생성된다. 또한 생성 시 단 한번만 작업을 진행한다.

   - 만약 액티비티가 계속 회전한다면? AsyncTask는 계속 쌓이게 될 것이다. 오래 걸리는 AsyncTask라면 끔찍하다.

3. 예외 처리 메소드 없음

   - 이 부분에서 AsyncTask를 정말 잘 처리해야한다고 생각한 부분이다. 

   - 이 부분은 조금의 구현을 통해서 callback을 달아주고 보다 디테일한 Thread 처리 기술이 될 수 있을 것이다.

     ```java
     public interface AsyncCallback<T> {
         public void onResult(T result);
     
         public void exceptionOccured(Exception e);
     
         public void cancelled();
     }
     public interface AsyncExecutorAware<T>{
     
         public void setAsyncExecutor(AsyncExecutor<T> asyncExecutor);
     }
     public class AsyncExecutor<T> extends AsyncTask<Void , Void , T> {
         private static final String TAG = "AsyncExecutor";
     
         private AsyncCallback<T> callback;
         private Callable<T> callable;
         private Exception occuredException;
     
         public AsyncExecutor<T> setCallable(Callable<T> callable) {
             this.callable = callable;
             return this;
         }
     
         public AsyncExecutor<T> setCallback(AsyncCallback<T> callback) {
             this.callback = callback;
             processAsyncExecutorAware(callback);
             return this;
         }
     
         @SuppressWarnings("unchecked")
         private void processAsyncExecutorAware(AsyncCallback<T> callback) {
             if (callback instanceof AsyncExecutorAware) {
                 ((AsyncExecutorAware<T>) callback).setAsyncExecutor(this);
             }
         }
     
         @Override
         protected T doInBackground(Void... params) {
             try {
                 return callable.call();
             } catch (Exception ex) {
                 Log.e(TAG,
                         "exception occured while doing in background: "
                                 + ex.getMessage(), ex);
                 this.occuredException = ex;
                 return null;
             }
         }
     
         @Override
         protected void onPostExecute(T result) {
             if (isCancelled()) {
                 notifyCanceled();
             }
             if (isExceptionOccured()) {
                 notifyException();
                 return;
             }
             notifyResult(result);
         }
     
         private void notifyCanceled() {
             if (callback != null)
                 callback.cancelled();
         }
     
         private boolean isExceptionOccured() {
             return occuredException != null;
         }
     
         private void notifyException() {
             if (callback != null)
                 callback.exceptionOccured(occuredException);
         }
     
         private void notifyResult(T result) {
             if (callback != null)
                 callback.onResult(result);
         }
     }
     ```

     [출처 : 유지보수를 고려한 안드로이드 비동기 처리 기반 코드 만들기](https://javacan.tistory.com/entry/maintainable-async-processing-code-based-on-AsyncTask)



### 마무리

AsyncTask는 **학습하고 처리를 올바르게 해준다면 문제 없이 잘 만들어진 기술**이라는 생각이 든다. 하지만, 편리하게 해주기 위해 만들어진 기술임에도 불구하고 불편하다.

AsyncTask를 유지보수할 수 있도록 생각한 코드나 작동 원리를 공부하는 것 등 개발자가 알아야 할 것이 많다. 이렇게 되면 처음에 말했던 Handler, Looper를 공부하는 것과 별반 다를게 없다고 생각하게 된다.

그리고 결과적으로 훌륭한 대안인 RxJava라는 Reactive 프로그래밍 패러다임이 나오면서 과거의 기술은 사라지고, 새로운 기술을 배워야하는 시기가 온 것이 아닌가 싶다. (그냥 싫다고 찡찡대는 것보다는 대안을 가지고 와서 문제점을 따진다면 보다 쉽게 바꿀 수 있으니까 말이다.)

## 참고문서

[Android AsyncTask](https://developer.android.com/reference/android/os/AsyncTask)

[Activitys, Threads, & Memory Leaks](https://www.androiddesignpatterns.com/2013/04/activitys-threads-memory-leaks.html)

[Can someone explain why is RxJava is better than AsyncTask? Preferably in plain English 😀](https://www.reddit.com/r/androiddev/comments/63f78o/can_someone_explain_why_is_rxjava_is_better_than/)

[AsyncTask와 비교해서 당신이 RxJava를 당장 써야하는 이유](http://ec2-54-180-165-79.ap-northeast-2.compute.amazonaws.com/2017/06/02/asynctask와-비교해서-당신이-rxjava를-당장-써야하는-이유/)