# 안드로이드 프레임워크 Part 2 - 안드로이드 이해에 필요한 기반 지식

Part 1에서 말했듯이 우리 안드로이드 개발자들은 안드로이드 프레임워크 위에서 동작하는 코드를 작성하기에 프레임워크를 이해하고자 해야할 것이다. 하지만 실상 이 모든 코드를 다 보는 것은 매우 힘들고, 이를 한 번에 다 보려는 것은 프로덕션 코드를 짜기 위해 필요한 지식 그 이상(어차피 안드로이드 앱을 만들기 위해서 안 필요한 지식이 어디있겠느냐마는)을 많은 시간을 투자하는 것과 동치이기에 이번 아티클에서는 안드로이드 내부 아키텍처의 Blue Print에 대해 적어보고자 한다.



## Android Architecture Overview

<img src="https://www.charlezz.com/wordpress/wp-content/uploads/2018/10/1262px-Android-System-Architecture.svg_.png" />



### 안드로이드 어플리케이션 (Applications)

일반적으로 우리가 사용하는 카톡, 페이스북, 인스타그램과 같은 일반앱과 운영체제 단에서 제공을 해주는 카메라, 전화, 브라우저와 같은 기본 앱(시스템 앱)은 어플리케이션 스택 위에서 동작한다.

이런 앱들은 안드로이드 프레임워크에서 제공하는 코드들/이를 활용하여 만든 오픈소스 라이브러리들을 활용하여 어플리케이션을 제작하여 사용자가 사용할 수 있도록 한다. 

#### Q) 시스템 앱과 일반 앱이 동일 계층에서 동작한다면 두 분류의 차이점은 무엇이 있는가?

시스템 앱과 같은 경우

- 시스템 권한을 사용할 수 있고
- 프로레스 자체의 우선순위를 시스템 단에서 높게 가져갈 수 있다

는 특징을 가지고 있다, 전화를 하는데 메모리가 부족하다고 해서 전화를 갑자기 종료를 시킬 수는 없지 않겠는가? 이런 논리로 시스템 앱들은 메모리가 부족한 상황이어도 가장 마지막에 메모리가 회수되게 시스템에서 그 우선순위를 정해준다.



### 어플리케이션 프레임워크(Application Framework)

어플리케이션 프레임워크는 어플리케이션을 만들 수 있게 해주는 기반이 되는 코드들이다. 기존에 안드로이드 코드를 작성해본 사람들은 알겠지만, Telephony Manager, Content Providers, Notification Manager 등이 이 Layer에 속한다.

이 Layer의 역할이 애매하다면 다음과 같은 상황을 생각해보자. 

+ 안드로이드에서 제공해주는 기본 화면 구성 단위인 Activity를 만들 때 생성자를 활용해서 직접 객체를 만들고 이를 호출한 적이 있는가?
+ ``R.string.some_string``에 들어있는 문자열 값을 가져올 때 이런 Resource File을 관리하는 클래스를 만들어본 적이 있는가?

위의 일들은 안드로이드 개발을 하면서 직접 작성을 해본적이 없을 것이다, 왜냐면 **이 역할을 어플리케이션 프레임워크에서 맡고 있기 때문**이다. 그래서 이 Layer는 시스템단에서 일어나는 일과 안드로이드 개발자 사이에서 발생하는 일련의 Flow를 숨겨주는 계층이라고 생각을 할 수 있을 것이다.



#### 프레임워크 컴포넌트들의 동작 방식

이런 컴포넌트들, 특히 안드로이드 4대 컴포넌트라 불리우는 Activity, Service, Broadcast Receiver, Content Provider들은 **Thin Client-Server 구조**로 컴포넌트의 동작을 관리한다.

여기서 클라이언트, 서버는 통상적으로 말하는 HTTP 통신을 활용하는 클라이언트-서버가 아닌

- 어떤 시스템 자원을 요청하고 이(활)용하는 Client
- 요청한 시스템 자원을 알맞게 제공하는 Server

를 의미한다. 즉 컴포넌트의 내부 동작도

- 상황에 맞게 적절한 함수(기능)을 호출하는 클라이언트(Thin Client)
- 이런 기능들을 처리하는 Server(``system_server``)
  - 단적으로 Activity라면 컴포넌트 탐색, Activity Stack 관리, ANR 처리 등이 있겠다
  - 위의 앱 프로세스와 별도로 운영된다

위와 같이 동작한다. 

예를 들어 ``startActivity()`` 함수를 호출할 때, ActivityManager에서 바로 액티비티를 탐색하는 것이 아니라 ``system_server``에서 해당 액티비티가 있는 지 요청하고 이 액티비티가 있으면 Activity Stack에 이를 반영하고 앱 프로세스에 액티비티를 띄우라고 메시지를 보낸다.

```java

    @Override
    public void startActivity(Intent intent, Bundle options) {
        warnIfCallingFromSystemProcess();
        // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
        // generally not allowed, except if the caller specifies the task id the activity should
        // be launched in. A bug was existed between N and O-MR1 which allowed this to work. We
        // maintain this for backwards compatibility.
        final int targetSdkVersion = getApplicationInfo().targetSdkVersion;
        if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
                && (targetSdkVersion < Build.VERSION_CODES.N
                        || targetSdkVersion >= Build.VERSION_CODES.P)
                && (options == null
                        || ActivityOptions.fromBundle(options).getLaunchTaskId() == -1)) {
            throw new AndroidRuntimeException(
                    "Calling startActivity() from outside of an Activity "
                            + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                            + " Is this really what you want?");
        }
        // startActivity 자체를 메인 스레드에서 작동시킴
        mMainThread.getInstrumentation().execStartActivity(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intent, -1, options);
    }

    /**
     * Logs a warning if the system process directly called a method such as
     * {@link #startService(Intent)} instead of {@link #startServiceAsUser(Intent, UserHandle)}.
     * The "AsUser" variants allow us to properly enforce the user's restrictions.
     */
    private void warnIfCallingFromSystemProcess() {
        if (Process.myUid() == Process.SYSTEM_UID) {
            Slog.w(TAG, "Calling a method in the system process without a qualified user: "
                    + Debug.getCallers(5));
        }
    }

    /**
     * Execute a startActivity call made by the application.  The default 
     * implementation takes care of updating any active {@link ActivityMonitor}
     * objects and dispatches this call to the system activity manager; you can
     * override this to watch for the application to start an activity, and 
     * modify what happens when it does. 
     *  
     * <p>This method returns an {@link ActivityResult} object, which you can 
     * use when intercepting application calls to avoid performing the start 
     * activity action but still return the result the application is 
     * expecting.  To do this, override this method to catch the call to start 
     * activity so that it returns a new ActivityResult containing the results 
     * you would like the application to see, and don't call up to the super 
     * class.  Note that an application is only expecting a result if 
     * <var>requestCode</var> is &gt;= 0.
     *  
     * <p>This method throws {@link android.content.ActivityNotFoundException}
     * if there was no Activity found to run the given Intent.
     * 
     * @param who The Context from which the activity is being started.
     * @param contextThread The main thread of the Context from which the activity
     *                      is being started.
     * @param token Internal token identifying to the system who is starting 
     *              the activity; may be null.
     * @param target Which activity is performing the start (and thus receiving 
     *               any result); may be null if this call is not being made
     *               from an activity.
     * @param intent The actual Intent to start.
     * @param requestCode Identifier for this request's result; less than zero 
     *                    if the caller is not expecting a result.
     * @param options Addition options.
     * 
     * @return To force the return of a particular result, return an 
     *         ActivityResult object containing the desired data; otherwise
     *         return null.  The default implementation always returns null.
     *  
     * @throws android.content.ActivityNotFoundException
     * 
     * @see Activity#startActivity(Intent)
     * @see Activity#startActivityForResult(Intent, int)
     * @see Activity#startActivityFromChild
     * 
     * {@hide}
     */
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    if (am.match(who, null, intent)) {
                        am.mHits++;
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess();
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
        }
        return null;
    }
```



### 안드로이드 런타임(ART, Android Runtime)

Java 코드를 바이트코드로 컴파일시키는 JVM과 같은 역할을 하는 가상 머신(Virtual Machine)이다.

일반적으로 자바는 여러 플랫폼에서 동일한 코드로 동작하는 것(**Write Once, Run Everywhere**)을 목적으로 만든 언어이기에, 운영체제가 아닌 가상 머신이라는 프로그램 위에서 바이트코드로 컴파일되어 어떤 운영체제에서든 개발 및 동작을 할 수 있게 만들어졌다.

그러나 이 JVM이 라이선스 문제가 있기에 구글에서는 

- 앱이 실행되고 나서 자주 사용되는 바이트 코드를 컴파일하는 방식인 **JIT(Just In Time)** 컴파일러인 Dalvik VM을 사용하다
- KitKat 버전 이후 성능 향상을 위해서 앱 설치를 할 때 모든 코드를 바이트코드로 변환하여 만드는 **AOT(Ahead Of Time)** 방식의 ART를 사용하고
- Android N 버전 이후 이 둘을 혼용하여 사용하는 하이브리드 형식의 가상머신을 사용하고 있다고 한다.



### 안드로이드 네이티브 라이브러리(Libraries)

안드로이드 운영체제는 모바일/임베디드 운영체제가 그 기원이었던 만큼 저용량(저전역)의 RAM/CPU 환경에서도 돌아갈 수 있어야 했기에 메모리 소비를 적게 만드는 C/C++ Native Library를 활용한다.

안드로이드에서 사용하는 대표적인 오픈소스 라이브러리는

- libc와 같은 시스템 C 라이브러리
- Surface Manager, Media Framework와 같은 Native System Service
- SGL(2D 그래픽), OpenGL, SQLite, WebKit과 같은 유틸 라이브러리

등이 있다.



### 안드로이드 커널(Linux Kernal)

안드로이드는 오픈소스 운영체제인 리눅스 커널을 기반으로 제작되었다. 개발자가 이쪽을 다루는 일은 거의 없다고 보면 되지만, 이 쪽의 역할에 대해서 알아보자면

- 하드웨어 추상화
- 메모리/전원 관리
- 보안 설정
- 네트워크 시스템/하드웨어 장치 드라이버 관리

등을 한다.



## 소결

안드로이드를 이해하기 위한 개요를 작성해보고 개발자가 정말 알아둬야할 계층들을 위주로 자세히 살펴보았다. 물론 이 글이 안드로이드의 모든 것을 알려주진 않는다, 하지만 이 글에 나와있는 내용들은 **안드로이드의 모든 것을 알기 위한 첫 걸음**이라고 자신있게 말할 수 있다. 정말로 방대하고 체감이 안될 정도로 까마득한 내용이지만 산티아고 순례길을 걷는 순례자처럼 하루하루 천천히 공부하다 보면 언젠가는 "안드로이드 고수"라는 목적지가 보이지 않을까?

