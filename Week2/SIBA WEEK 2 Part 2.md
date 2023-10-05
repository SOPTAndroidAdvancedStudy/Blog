# 메인 스레드와 Handler Part2 -  안드로이드 애플리케이션에서의 메인 스레드

> 안드로이드 애플리케이션에서  Handler와 Looper 그리고 MessageQueue 구조가 어떻게 쓰이는지 알아봅니다.

앞서 Part 1에서 알아보았던, Handler와 Looper 구조와 동작이 과연 왜 필요할까요? 

이는 여러 스레드를 사용하는 멀티스레드 환경에서, 특정 스레드에게 일을 위임하기 위한 수단으로 의미가 있습니다. 백그라운드 스레드가 특권을 가진 메인스레드에게 일을 분담하기 위해, 기본적으로 사용되는 구조가 되는 것입니다. 

이번 아티클에서는 과연 그 '특권'이란 무엇이고 왜 생겨났는지, 어떻게 사용되어야하는지에 대해 알고자 합니다. 안드로이드 프레임 워크의 관점에서의 메인 스레드의 의미와 역할에 대해 알아보겠습니다. 

## 안드로이드의 메인 스레드

### 안드로이드의 메인 스레드가 하는 일

- 안드로이드의 메인스레드를 찾기 위해, 일반적인 프로그램의 시작점인 main()를 찾아보도록 하겠습니다. Android FW의 main()는 [ActivityThread.java](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/ActivityThread.java;l=247?q=ActivityThread&sq=&hl=ko) 에서 찾을 수 있습니다. 

  ```java
  public static void main(String[] args) {
          Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
  
          // Install selective syscall interception
          AndroidOs.install();
  
          // CloseGuard defaults to true and can be quite spammy.  We
          // disable it here, but selectively enable it later (via
          // StrictMode) on debug builds, but using DropBox, not logs.
          CloseGuard.setEnabled(false);
  
          Environment.initForCurrentUser();
  
          // Make sure TrustedCertificateStore looks in the right place for CA certificates
          final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
          TrustedCertificateStore.setDefaultUserDirectory(configDir);
  
          // Call per-process mainline module initialization.
          initializeMainlineModules();
  
          Process.setArgV0("<pre-initialized>");
  
          Looper.prepareMainLooper();
  
          // Find the value for {@link #PROC_START_SEQ_IDENT} if provided on the command line.
          // It will be in the format "seq=114"
          long startSeq = 0;
          if (args != null) {
              for (int i = args.length - 1; i >= 0; --i) {
                  if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                      startSeq = Long.parseLong(
                              args[i].substring(PROC_START_SEQ_IDENT.length()));
                  }
              }
          }
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
  ```

- 안드로이드의 메인스레드는 Activity, Service, Broadcast Reciver, Application와 같은 컴포넌트 생명주기 메서드와 관련된 작업들을 기본적으로 담당하고 있습니다. 

### UI 작업에는 단일 스레드 모델을 적용

- UI 작업에 필요한 UI 자원 공유를 하면서 발생할 수 있는 경합 상태, 교착 상태를 방지하고자, **메인스레드의 UI 작업에는 단일 스레드 모델**이 적용됩니다. 

> 단일 스레드 모델은 자원 접근에 대한 동기화를 신경쓰지 않아도 되고, 작업전환(context switching) 비용을 요구하지 않으므로, 경합 상태와 교착 상태를 방지할 수 있다.

- 즉, 안드로이드에서의 단일 스레드 모델이란 안드로이드 화면을 구성하는 뷰나 뷰그룹을 하나의 스레드(메인 스레드)에서만 담당하는 원칙을 말합니다. 단일 스레드 모델은 아래 두 가지 규칙을 갖습니다.

  > 첫째, 메인 스레드(UI 스레드)를 블럭하지 말 것
  >
  > 둘째, 안드로이드 UI 툴킷은 오직 UI 스레드에서만 접근할 수 있도록 할 것

### 메인스레드와 백그라운드 스레드 통신을 위한 Handler와 Looper 구조

- 단일 스레드에서의 긴 작업은 어플리케이션의 반응성을 낮추거나, ANR의 원인이 될 수 있습니다. 따라서 메인 스레드에선 정해진 최소한의 일만 담당하고, 특히 긴 작업은 다른 스레드가 담당하게 해야합니다. 
- 따라서 이 메인 스레드와 다른 스레드가 협업하기위해, **스레드간 통신**이 필요하게 되었습니다.
- 안드로이드에선 **Looper와 Handler를 사용하여, 다른 스레드와 메인 스레드간의 통신**을 할 수 있습니다.
- 위 내용을 정리하면, **안드로이드에서의 Thread-Looper-Handler 구조는, 메인 스레드에 단일 스레드 모델이 적용되면서 요구되는 스레드간의 통신 방법을 지원하는 구조**라고 이해하면 좋습니다.

## Handler 사용 예시 

### 백그라운드 스레드에서 UI 작업

- 앞서 설명 드렸듯 UI 작업은 단일 스레드 모델이 적용되어 메인 스레드에서만 실행됩니다. 따라서 네트워크 통신 같은 백그라운드 작업을 하면서 발생하는 UI 변경 작업은 Handler 사용하여 메인 스레드로 위임되고, 메인 스레드에서 실행됩니다. 

### 반복 UI 갱신를 위한 recursive Runnable

- 시계나 타이머처럼 반복적으로 UI 작업을 해야할 때가 있습니다. 이 UI 갱신 작업은 Runnable을 재귀적으로 설계하는 것으로 구현할 수 있습니다. 

  ```kotlin
  package com.example.myapplication
  
  import android.os.Bundle
  import android.os.Handler
  import android.widget.TextView
  import androidx.appcompat.app.AppCompatActivity
  
  
  class MainActivity : AppCompatActivity() {
      private val DELAY_TIME = 2000L
      private val handler = Handler(mainLooper)
      private var updateTimeRunnable: Runnable? = null
  
      override fun onCreate(savedInstanceState: Bundle?) {
          super.onCreate(savedInstanceState)
          setContentView(R.layout.activity_main)
          updateTimeRunnable = Runnable {
              findViewById<TextView>(R.id.tv1).text = System.currentTimeMillis().toString()
              updateTimeRunnable?.let { handler.postDelayed(it, DELAY_TIME) }
          }
  
      }
  
  
      fun onClickButton(view: TextView){
          updateTimeRunnable?.let { handler.post(it) }
      }
  
  }
  
  ```

  

### 타이머, ANR 판단 

- 

## 요약

- 안드로이드의 메인 스레드는 [ActivityThread.java](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/ActivityThread.java;l=247?q=ActivityThread&sq=&hl=ko) 에서 시작되며, 컴포넌트 생명주기 메서드와 관련된 작업들을 담당합니다.
- 경합 상태, 교착 상태를 방지하고자, UI 작업에는 단일 스레드 모델을 적용합니다. 따라서 UI 작업은 메인 스레드에서만 실행 가능합니다.
- UI 작업은 메인 스레드에서만 실행 가능하기 때문에, 작업에 따라 메인 스레드와 백그라운드 스레드간의 통신이 필요합니다. Handler와 Looper 구조는 스레드간 통신을 위한 구조 입니다.
- Handler 사용 (스레드간 통신)의 예시로는 백그라운드 스레드에서 UI 작업, UI 갱신, 시간 제한 작업이 있습니다. 
