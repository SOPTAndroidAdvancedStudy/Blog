# 안드로이드 프레임워크 Part 1 - 안드로이드를 어떻게 공부해야할 것인가 

많은 개발자들은 개발을 하다 막히는 지점이 있을 때 [스택 오버플로우(Stack Overflow)](https://stackoverflow.com/)에 존재하는 질문/답변들을 참고하여 문제를 해결하려고 하는 경향이 짙습니다(물론 이런 말을 하는 저도 예외는 아닙니다). 하지만 이에 심하게 의존할 경우 현재 겪고 있는 상황이 질문의 상황과 부합하지 않거나 잘못된 답안이 올라왔음에도 일말의 의심 없이 이를 차용하기에 다시 문제 상황에 봉착할 수 있습니다. 

이런 상황에 다시 걸려들지 않으려면, 또 문제를 풀어나가면서 개발 실력을 향상시킬 수 있으려면 어떻게 문제를 해결해나가야 할까요?

## 개발에서 "검증"의 필요성

안드로이드 어플리케이션은 **어플리케이션 프레임워크**을 활용하여 제작합니다. **프레임워크**라는 것은 안드로이드 어플리케이션을 제작할 때 프레임워크에서 정해준 동작(규칙)대로만 만들기만 하면 이를 수행하는 하나의 위임체라고 생각할 수 있습니다. 

그렇다면 안드로이드 개발을 할 때, 어떤 문제에 봉착했다는 것은 **내가 모르는 "프레임워크가 정한 규칙"**이 있다는 것이고, 이를 해결하기 위해서는 이 규칙에 맞춰서 틀린 코드를 수정해나가야 할 것입니다. 

**여기서 이 상황을 해결하기 위한 코드만 알아가고 이에 대한 규칙을 이해하지 못한다면 어떻게 될까요?** 얼마 전, 제가 경험했던 일로 프레임워크 레벨단 코드를 보면서 문제를 해결해나가는 것이 얼마나 중요한 것인지 설명해보도록 하겠습니다.

제가 맡았던 태스크 중에서 화면 캡쳐를 막아야하는 기능을 구현해야하는 것이 있었습니다. 저는 이 문제의 해결책을 마련하기 위해 바로 Google에 "android prevent screen capture"를 검색했고 다음과 같은 답변을 StackOverflow에서 얻을 수 있었습니다.

> To disable Screen Capture:
>
> Add following line of code in `onCreate()` method:
>
> ```scss
> getWindow().setFlags(WindowManager.LayoutParams.FLAG_SECURE,
>                            WindowManager.LayoutParams.FLAG_SECURE);
> ```

아 그렇다면, 저는 여기서 ``onCreate()`` 함수의 바로 다음줄에 적으면 되겠구나라고 생각을 해서 바로 다음과 같이 코드를 작성했고 이는 성공적으로 실행이 되었습니다.

```kotlin
    override fun onCreate(savedInstanceState: Bundle?) {
        window().setFlags(WindowManager.LayoutParams.FLAG_SECURE,
                WindowManager.LayoutParams.FLAG_SECURE)
        super.onCreate(savedInstanceState)
        binding = DataBindingUtil.setContentView(this, R.layout.activity_sample)
```

하지만 추가 요청사항으로 Hilt(Android에서 사용하는 의존성 주입 프레임워크)로 주입된 객체를 활용하여 조건문을 만들었을 때 ``UninitializedPropertyAccessException``과 함께 앱 크래쉬가 발생했습니다. 

```kotlin
    override fun onCreate(savedInstanceState: Bundle?) {
        if (controlManager.isAvailable) {
          window().setFlags(WindowManager.LayoutParams.FLAG_SECURE,
                WindowManager.LayoutParams.FLAG_SECURE)
        }
        super.onCreate(savedInstanceState)
        binding = DataBindingUtil.setContentView(this, R.layout.activity_sample)
```

이때 위의 문제점이 반쪽짜리 문제점임을 파악, 우선 아래와 같이 문제를 접근했습니다.

> 우선 Hilt로 생성된 객체로 인해서 크래쉬가 발생했을 가능성이 높다 이에 대해서 크래쉬를 해결하자

그랬더니 Hilt를 통해 생성된 ``Hilt_SampleActivity``와 ``AppCompatAcitvity``의 Parent 클래스인 ``ComponentActivity``에서 다음과 같은 코드를 발견할 수 있었습니다.

```kotlin
  // Hilt_SampleActivity
  Hilt_SampleActivity(int contentLayoutId) {
    super(contentLayoutId);
    _initHiltInternal();
  }

  private void _initHiltInternal() {
    addOnContextAvailableListener(new OnContextAvailableListener() {
      @Override
      public void onContextAvailable(Context context) {
        // Injection Function
        inject();
      }
    });
  }

  // ComponentActivity
    public ComponentActivity() {
        /* 중략 */
        addOnContextAvailableListener(new OnContextAvailableListener() {
            @SuppressLint("SyntheticAccessor")
            @Override
            public void onContextAvailable(@NonNull Context context) {
                Bundle savedInstanceState = getSavedStateRegistry()
                        .consumeRestoredStateForKey(
                                ACTIVITY_RESULT_TAG);
                if (savedInstanceState != null) {
                    mActivityResultRegistry.onRestoreInstanceState(savedInstanceState);
                }
            }
        });
    }
```

``SampleActivity``는 Hilt를 통해 바이트코드 단에서 ``Hilt_SampleActivity``와 동일하게 동작하고 내부 코드 중 ``_initHiltInternal``이란 함수에서 ``addOnContextAvailableListener``란 함수를 작동시킨다는 것을 알 수 있었습니다. 이는 ``ComponentActivity``에서 어떻게 동작하는 지 javadoc을 통해 알 수 있었습니다.

>  addOnContextAvailableListener 클래스를 통해 추가된 OnContextAvailableListener 콜백은 context가 생성되기 이전에 추가되고, 이 순서대로 동작을 하게 됩니다. 이 콜백들은 super.onCreate()의 일부로 동작하게 됩니다.

즉, Hilt로 Activity에 주입된 객체들은 ``super.onCreate`` 이후에 사용을 할 수 있었다는 사실을 알게되었습니다. 

그러나 이 조건문을 옮기면 캡쳐 방지 기능이 제대로 동작하지 않을 것 같아, ``super.onCreate`` 이후의 어느 곳에 위치를 시켜야할 지 몰라 이번에는 ``setFlags`` 함수를 조사하게 되었고, javadoc에서 일부 flag는 반드시 ``setContentView`` 함수 이전에 설정해야하는 javadoc을 보고 ``DataBindingUtil.setContentView`` 함수를 조사하게 되었습니다.

```java
    // DataBindingUtil.java
    public static <T extends ViewDataBinding> T setContentView(@NonNull Activity activity,
            int layoutId, @Nullable DataBindingComponent bindingComponent) {
        activity.setContentView(layoutId);
        View decorView = activity.getWindow().getDecorView();
        ViewGroup contentView = (ViewGroup) decorView.findViewById(android.R.id.content);
        return bindToAddedViews(bindingComponent, contentView, 0, layoutId);
    }

    // Activity.java
    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }

    // Winodw.java
    /*
     * Note that calling this function "locks in" various characteristics
     * of the window that can not, from this point forward, be changed: the
     * features that have been requested with requestFeature(int),
     * and certain window flags as described in setFlags(int, int).
     */
    public abstract void setContentView(View view, ViewGroup.LayoutParams params);
```

즉 ``DataBindingUtil.setContentView()``은 ``Activity``의 ``setContentView()`` 호출하고, 이는 다시 ``Window``의 ``setContentView()``를 호출하게 되는데, 이는 ``setFlags()``에서 지정해둔 flag들을 활용하여 ``window``의 feature를 결정해준다고 서술되어 있습니다. 즉, 위의 코드들은 ``super.onCreate()``와 ``DataBindingUtil.setContentView()`` 사이에 위치하면 정상적인 작동이 보장된다는 것이었고 실제로 아래와 같이 코드를 작성하니 화면 캡쳐 방지 기능이 원하는 대로 구현되었습니다.

## 소결

위에서 제가 작성했던 글이 "안드로이드를 제대로 공부하려면 스택오버플로우는 버리고 프레임워크 단 코드만 보고 공부해라"라는 의미로 전달되었다면, 이는 오해라고 말하고 싶습니다. 프레임워크 레벨의 코드는 안드로이드를 처음 공부하는 사람들에게도, 저에게도, 심지어 저보다 오래 공부한 개발자들조차 어려워합니다. 이 코드만 분석해서 공부해라라고 하는 것은 어쩌면 굉장히 "비효율적인" 행위로도 생각하고 있습니다.

하지만, 복잡하고 정교한 기능을 사용자가 사용할 수 있게 구현해야하는 Android Engineer가 되려면 스택 오버플로우, Github에 있는 코드를 볼 뿐만 아니라, 문제의 발생/해결 지점을 내부에서 해결할 수 있는 능력도 갖춰야한다는 것을 이 글을 통해서 공유하고 싶습니다. 

