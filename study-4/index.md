#### 7장
- 작업중단 
```text
다음의 경우 보통 작업 중단이 일어난다
- 사용자가 취소하기를 요청한 경우
- 시간이 제한된 작업
- 애플리케이션 이벤트
- 오류 & 종료

취소정책(cancellation policy)
작업을 쉽게 취소하기 위하여 어떻게, 언제, 어떤 것을 해야하는지 정의 할 필요가 있다
```

인터럽트 정책: 작업을 실행하는 스레드의 인터럽트 상태에 대하여 대응이 가능하도록 해야한다
- InterruptedException 대응법:
  호출 스택 상위로 전달
  ```java
  public Task getNextTask() throws InterruptedException {
    return queue.take();
  } 
  ```
  정책에 따라 대응한다
  ```java
  public Task getNextTask() throws InterruptedException {
    try {
        // do something
    } catch (InterruptedException e){ // 정해진 정책에 의한 처리이므로 아무것도 하지 않아도 무방하다
        // continue
    }
  } 
  ```
- Future 사용
```java
public static void timedRun(Runnable r, long timeout, TimeUnit unit) {
  Future<?> task = taskExec.submit(r); 
  try {
    task.get(timeout, unit);
  } catch (TimeoutException e) {
      // finally 블록에서 중지 될 것
  } catch (ExecutionException e) {
    throw e;
  } finally {
    // 실행중이라면 인터럽트 수행
    task.cancel(true);
  }
}
```

- 스레드기반 서비스 중단  
ExecutorService 활용하자
```java
public abstract class WebCrawler {
  private volatile TrackingExecutor exec;
  @GuardedBy("this")
  private final Set<URL> urlsToCrawl = new HashSet<URL>();
    ...

  public synchronized void start() {
    exec = new TrackingExecutor(Executors.newCachedThreadPool());
    for (URL url : urlsToCrawl) submitCrawlTask(url);
    urlsToCrawl.clear();
  }

  public synchronized void stop() throws InterruptedException {
    try {
      saveUncrawled(exec.shutdownNow());
      if (exec.awaitTermination(TIMEOUT, UNIT)) // <---
        saveUncrawled(exec.getCancelledTasks()); // <---
    } finally {
      exec = null;
    }
  }
  
  ...
}
```
- 비정상적인 스레드 종료 상황 처리 
UncaughtException 처리를 구현하여 얘외에 대응할 수 있도록 하자
```java
public interface UncaughtExceptionHandler {
    void uncaughtException(Thread t, Throwable e);
}

public class UEHLogger implements Thread.UncaughtExceptionHandler {
  public void uncaughtException(Thread t, Throwable e) {
    Logger logger = Logger.getAnonymousLogger();
    logger.log(Level.SEVERE, "Thread terminated with exception: " + t.getName(), e);
  }
}
```
- jvm 종료
```text
  - 크게 두가지 경우가 있다 
    - 예정된 경우와 그렇지 않은 경우
  - 스레드는 크게 두가지로 나눠 볼 수 있다
    - jvm 내부적으로 사용하는 데몬 스레드(gc 등) 그리고 메인 스레드에서 생성한 일반 스레드
  - finalizer & cleaner 사용을 피하자: item 8, effective java
```