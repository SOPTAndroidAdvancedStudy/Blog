# WorkManager - For Long Period Task

## Motivation

> Asynctask was intended to enable proper and easy use of the UI Thread.
> AsyncTasks should ideally be used for short operations (a few seconds at the most.) If you need to keep threads running for long periods of time, it is highly recommended you use the various APIs provided by the `java.util.concurrent` package such as `Executor`, `ThreadPoolExecutor` and `FutureTask`.

AsyncTask는 짧은 시간에 수행되는 태스크들을 백그라운드에서 돌려야할 때 사용되었다. 비록 이제는 Deprecated 되어서 RxJava, Coroutine으로 대체되었지만.

그렇다면 백그라운드에서 장시간 태스크를 수행해야한다면 어떤 것을 활용해야할까?

## Before WorkManager

### Build.SDK_INT < 19 (KitKat)

AlarmManager와 BroadcastReceiver를 활용하여 만든다. 

```java
public class SchedulerSetupReceiver extends BroadcastReceiver {
	private static final String APP_TAG = "siba.android";
 
	private static final int EXEC_INTERVAL = 20 * 1000;
 
	@Override
	public void onReceive(final Context context, final Intent intent) {
		Log.d(APP_TAG, "SchedulerSetupReceiver.onReceive() called");
		AlarmManager alarmManager = (AlarmManager) context
				.getSystemService(Context.ALARM_SERVICE);
		Intent receiverIntent = new Intent(context, SchedulerEventReceiver.class);
		PendingIntent intentExecuted = PendingIntent.getBroadcast(context, 0, receiverIntent,
				PendingIntent.FLAG_CANCEL_CURRENT);
		Calendar now = Calendar.getInstance();
		now.add(Calendar.SECOND, 20);
		alarmManager.setRepeating(AlarmManager.RTC_WAKEUP,
				now.getTimeInMillis(), EXEC_INTERVAL, intentExecuted);
	}
}
```

AlarmManager: 특정 시간에 어떤 코드를 수행하기 위해서 작성된 코드 위의 동작은 20초마다 특정 작업을 수행하기 위해서 ``alarmManager.setRepeating`` 함수를 사용한다.

하지만 API 19에서 AlarmManager는 정확히 특정 시간에 알람이 오지 않고 특정 시간에 알람이 몰아서 오거나 지연 처리 등의 이슈가 있어서 백그라운드 태스크를 작업하는데 애로사항이 존재했었다.

### Build.SDK_INT >= 21 (Lolipop)

이런 부정확성을 해결하기 위해서 안드로이드에서는 JobScheduler라는 새로운 백그라운드 태스크 처리 API를 내놓았다.

기존 알람매니저는 특정 주기마다 핸드폰이 어떤 상황이든 관계 없이 동작시켰다면, 잡스케줄러는 특정상황(디바이스가 충전 중이고 핸드폰을 만지지 않는 상황(IDLE))에서만 백그라운드 태스크를 작업할 수 있게 만들었고 비주기적인(1회성) 백그라운드 태스크도 작동할 수 있게 만들었다.

```kotlin
class MyJobService : JobService() {
    companion object {
        private val TAG = "MyJobService"
    }

    override
    fun onStartJob(params: JobParameters): Boolean {
        Log.d(TAG, "onStartJob: ${params.jobId}")

        thread(start = true) {
            Thread.sleep(1000)
            Log.d(TAG, "doing Job in other thread")
            jobFinished(params, false)
        }

        return true
    }

    override
    fun onStopJob(params: JobParameters): Boolean {
        Log.d(TAG, "onStopJob: ${params.jobId}")
        return false
    }
}


        btnJob1.setOnClickListener {
            val js = getSystemService(Context.JOB_SCHEDULER_SERVICE) as JobScheduler
            val serviceComponent = ComponentName(this, MyJobService::class.java)
            val jobInfo = JobInfo.Builder(JOB_ID_A, serviceComponent)
                // 최소한 이때는 작업이 수행된다
                .setMinimumLatency(TimeUnit.MINUTES.toMillis(1))
                // 이때까지 못 끝내면 도게자한다
                .setOverrideDeadline(TimeUnit.MINUTES.toMillis(3))
                .build()
            js.schedule(jobInfo)
            Log.d(TAG, "Scheduled JobA")
        }

        btnJob2.setOnClickListener {
            val js = getSystemService(Context.JOB_SCHEDULER_SERVICE) as JobScheduler
            val serviceComponent = ComponentName(this, MyJobService::class.java)
            val jobInfo = JobInfo.Builder(JOB_ID_B, serviceComponent)
                // 사용자가 폰을 사용하지 않아야됨
                .setRequiresDeviceIdle(true)
                // 반드시 충전중이어야됨
                .setRequiresCharging(true)
                .setPeriodic(TimeUnit.MINUTES.toMillis(15))
                // 주기적으로 해야됨
                .build()
            js.schedule(jobInfo)
            Log.d(TAG, "Scheduled JobB")
        }
```

제약조건들

- setRequiresDeviceIdle
- setRequiresCharging
- setRequiredNetworkType
- setRequiredNetwork
- setRequiresBatteryNotLow
- setRequiresStorageNotLow

### API 26의 비극

Background Service의 제약 - App 이 Background 일 때 Foreground Service 가 아니면 Background Service 사용 불가, 백그라운드 서비스를 동시에 여러 앱이 이용하면서 메모리, 배터리 리소스를 마음대로 사용하여 폰 성능을 굉장히 저하시켜 왔었음

구글에서 이런 문제를 해결하기 위해 Google Services에서 Firebase JobDispatcher라는 것을 새로 만들어서 제공, but 이는 그다지 좋은 해결책이 되지 못함(Google Service, 써드파티에 의존성이 생김)

## WorkManager Resolve This

결국에는 개발자는 API 21 미만이면 AlarmManager + BroadcastReceiver 조합을, Firebase 의존성이 있다면 Firebase JobDispatcher를 아니면 JobScheduler로 구현된 코드를 모두 가지고 있어야 했습니다. 그러나 WorkManager는 이런 백그라운드 프로세스를 내부에 처리하고 그 과정을 추상화함으로써 개발자로 하여금 한 API에서만 처리할 수 있게 도와줍니다.

![img](https://miro.medium.com/max/1400/1*hpsHT-TCsO3px3Q72-7Nvg.png)

![img](https://miro.medium.com/max/1400/1*9WbyMPcUcMq65UDuoDdktg.png)

(여기서 JobScheduler에 위치한 포지션을 대체했다고 생각하면 됨)

즉 WorkManager는 실행 보장이 가능하고, 장치 상태에 따라서 태스크를 중단/재실행이 가능한 작업 관리자 역할을 한다. 또한 특정 작업이 완료되면 다른 작업을 진행할 수 있거나 여러 작업들을 동시에 처리할 수 있다(병렬처리).

![img](https://miro.medium.com/max/1400/1*gD2eOZthfr2d5dqZLdHZqw.png)

## How to use WorkManager

### WorkManager의 구성

- WorkManager: WorkQueue에 작업들을 넣고 처리한다. 자바에서 BlockingQueue 같은 느낌으로다가
- Worker: 이 추상클래스 내부에 doWork라는 함수를 오버라이드 해서 어떤 작업을 할 것인지, 작업의 결과물을 어떻게 배출할 것인지 결정
- WorkRequest: Worker의 작업 여부(반복/비주기, 제약사항, 실행조건 등)를 결정하는 클래스.
  - OneTimeWorkRequest
  - PeriodicWorkRequest

```kotlin
class SimpleWorker : Worker() {
    override fun doWork(): Result {
        Log.d("Sample", "SimpleWorker Working...")
        return Result.SUCCESS
    }
}

    // 작업을 수행하는 클래스 내부
    private fun doWorkOneTime() {
        val workRequest = OneTimeWorkRequestBuilder<SimpleWorker>().build()
        val workManager = WorkManager.getInstance()
        workManager?.enqueue(workRequest)
    }

    private fun doWorkPeriodic() {
        val workRequest = PeriodicWorkRequestBuilder<SimpleWorker>(15, TimeUnit.MINUTES).build()
        PeriodicWorkRequest.Builder(SimpleWorker::class.java, 15, TimeUnit.MINUTES).build()
        val workManager = WorkManager.getInstance()
        workManager?.enqueue(workRequest)
    }

    private fun doWorkWithConstraints() {
        // 네트워크 연결 상태 와 충전 중 인 상태를 제약조건으로 추가 한다
        val constraints = Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .setRequiresCharging(true)
                .build()

        // 제약 조건과 함께 작업 요청 생성
        val requestConstraint  = OneTimeWorkRequestBuilder<SimpleWorker>()
                .setConstraints(constraints)
                .build()

        val workManager = WorkManager.getInstance()

        workManager?.enqueue(requestConstraint)
    }

    private fun doWorkChaining() {
        val compressWork = OneTimeWorkRequestBuilder<CompressWorker>().build()
        val uploadWork = OneTimeWorkRequestBuilder<UploadWorker>().build()

        WorkManager.getInstance()?.apply {
            beginWith(compressWork).then(uploadWork).enqueue()
        }
    }

    private fun doWorkChaining2() {
        // 세개의 필터적용 작업 요청 생성
        val filterWork1 = OneTimeWorkRequestBuilder<FilterWorker>().build()
        val filterWork2 = OneTimeWorkRequestBuilder<FilterWorker>().build()
        val filterWork3 = OneTimeWorkRequestBuilder<FilterWorker>().build()

        val compressWork = OneTimeWorkRequestBuilder<CompressWorker>().build()
        val uploadWork = OneTimeWorkRequestBuilder<UploadWorker>().build()

        WorkManager.getInstance()?.apply {
            beginWith(filterWork1, filterWork2, filterWork3).then(compressWork).then(uploadWork).enqueue()
        }
    }
```

### 여러 Worker에서 오는 결과물을 처리해야할 때? InputMerger

![img](https://miro.medium.com/max/1400/1*UOT1_rHQndtKNBonXdL_mA.png)

책의 단어 빈도를 카운트하고 이를 빈도순대로 정렬하는 작업을 한다고(이걸 왜 핸드폰으로 하지) 가정을 해보자. 그렇다면 책1, 책2, 책3에서 카운트 한 Work의 결과물들이 다음 Worker의 ``inputData`` 로 올텐데 이런 경우 WorkManager에서 어떻게 처리할까?

```kotlin
private fun doWorkChaining() {
        // 전달할 정보를 담은 Map 객체를 생성합니다.
        val input = mapOf("file_name" to "sdcard/user_choice_picture.jpg")

        // Data 클래스의 Builder 를 사용해서 input 을 담고 있는 Data 객체를 생성합니다.
        val inputData = Data.Builder().putAll(input).build()

        // 코틀린의 경우 Map 에 제공되는 인라인 함수 toWorkData() 를 이용해 쉽게 Data 객체를 생성할수도 있습니다.
//        val inputData = input.toWorkData()

        // WorkRequest 의 setInputData() 메서드로 작업자에 정보를 전달합니다.
        val compressWork = OneTimeWorkRequestBuilder<CompressWorker>()
                .setInputData(inputData)
                .build()

        val uploadWork = OneTimeWorkRequestBuilder<UploadWorker>().build()

        WorkManager.getInstance()?.apply {
            beginWith(compressWork).then(uploadWork).enqueue()
        }
}

class CompressWorker : Worker() {
    override fun doWork(): Result {
        // Data로 들어온 것
        val fileName = inputData.getString("file_name", "")

        Log.d("Sample", "CompressWorker Target : ${fileName!!}")

        // 전달받은 파일이름의 이미지를 압축합니다.

        // 결과물을 compress.jpg 로 저장합니다.

        // 이제 compress.jpg 를 UploadWorker 로 전달합니다.
        outputData = mapOf("compress_file_name" to "compress.jpg").toWorkData()

        return Result.SUCCESS
    }
}


class UploadWorker : Worker() {
    override fun doWork(): Result {
        val fileName = inputData.getString("compress_file_name", "")

        Log.d("Sample", "UploadWorker Target : ${fileName!!}")

        return Result.SUCCESS
    }
}
```

(우선 위와 같이 Data를 Worker들기리 전달을 한다고 한다)

![img](https://miro.medium.com/max/1400/1*HNC_lxA4SWob02V9nmpDHA.png)

OverwritingInputMerger 는 여러개의 Data 가 전달될때 같은 key 를 가지는 value 는 덮어쓴다(Overwrite) 데이터 간에 중복 되지 않는 key 의 value 는 새롭게 추가하여 하나의 Data 객체를 만든다.

![img](https://miro.medium.com/max/1400/1*pcvuPKt2sDetpaIP-sbvVA.png)

ArrayCreatingInputMerger 는 여러개의 Data 가 전달될때 같은 key를 가지는 value를 배열로 전달합니다.

```kotlin
val sortWordWorker = OneTimeWorkRequestBuilder<SortWorker>()
                .setInputMerger(ArrayCreatingInputMerger::class)
                .build()
```

