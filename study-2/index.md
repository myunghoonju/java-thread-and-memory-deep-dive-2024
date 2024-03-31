#### 4장
스레드 안전한 클래스 설계  
> 고려사항
> - 객체의 상태를 보관하는 변수는 무엇인가?  
> - 이 변수가 가질 수 있는 값이 어떠한 종류, 범위에 해당하는가?   
> - 객체 내부의 값을 동시에 사용할 때 **관리 정책**은 무엇인가?
>  - 동기화 정책
>    - 여러 스레드가 동시에 접근하여 객체의 변수가 유동적으로 변하는 상황에서도  
>      값을 안전하게 사용할 수 있도록 조절하는 방법 
>    - 정책의 종류: 객체의 불변성, 스레드 한정, 락  


>  동기화 요구사항  
> - 서로 연관된 값은 단일 연산으로 한번에 읽거나 변경해야 올바른 상태를 유지하기 용이하다  
> - 객체가 가질 수 있는값의 범위와 변동 폭을 정확하게 알아야 한다  
> - 제약조건이 다수인 경우 상태 변수를 클래스 내부에 은닉하거나 추가적인 동기화 기법이 필요하다  

> 상태 의존 연산
>   - 조건에 따라 동작 여부가 결정되는 연산  

> 상태 소유권
> 자바의 경우 가비지 컬렉터가 객체 공유와 관련된 오류가 발생하기 쉬운 부분을 조절해준다  
> 객체를 외부로 공개하면 공동 소유권을 가지는 셈이다  
> 컬렉션 클래스의 경우 **소유권 분리**의 형태가 존재한다
>   - 예시: ServletContext
>     - 추가된 객체들을 보관만 하고 있을 뿐이다
>     - 호출해서 사용하는 곳에서 소유권을 가지고 있다

인스턴스 한정  
- 객체를 적절하게 캡슐화하는 것으로 안정성을 확보하는 것  
- 데코레이터 패턴을 활용한 팩토리 메소드가 적용된 Wrapper 클래스가 좋은 예시이다
- 이 클래스를 거쳐야만 원래 컬렉션 클래스의 내용을 사용할 수 있기 때문이다(직접접근 X)
- 인스턴스를 한정함은 자바 모니터 패턴에서 찾아 볼 수 있다
```java
public class PrivateLock {
    private final Object myLock = new Object();
    
    @GuardedBy("myLock")
    Widget widget;
    
    void someMethod() {
        synchronized (myLock) {
            // widget 값 읽거나 변경한다
        }
    }
}
```
스레드 안정성 위임
- 스레드 안전한 다른 클래스에게 역할을 위임하고 그 외에는 상태를 가지지 않는 경우이다
```java
import java.util.Collections;
import java.util.HashMap;
import java.util.concurrent.ConcurrentHashMap;

public class DelegatingVehicleTracker {
    private final ConcurrentHashMap<String, Point> location; // 스레드 안전성 보장
    private final Map<String, Point> unmodifiablemap; // 스레드 안전성 보장
    
    public Map<String, Point> location() {
        return Collections.unmodifiableMap(new HashMap<>(this.location));
    }
}
```
- 위임할 때 문제점
  - 두 개 이상의 변수를 사용하는 경우 복합 연산 시 외존성 조건을 지키지 못하는 경우가 발생
  - A thread setLower(5), B thread setUpper(4) 호출하면?! 조건에 어긋난다
```java
public class NumberRange{
    // 의존성 조건:lower <= upper
    private final AtomicInteger lower = new AtomicInteger(0);
    private final AtomicInteger upper = new AtomicInteger(0);

    public void setLower(int i){
        // 주의 안전하지 않는 비교문
        if(i> upper.get()) {
          throw new IllegalArgumentException("can'set lower to "+i+"> upper");
        }
        
        lower.set(i);
    }

    public void setUpper(int i){
        // 주의 - 안전하지 않은 비교문
        if(i < lower.get()) {
          throw new IllegalArgumentException ("can't set upper to "+i+"< lower");
        }
        
        upper.set(i);   
    }

    public boolean isInRange(int i){
        return (i>= lower.get() && i <= upper.get());
    }
}
```
스레드 안전하게 구현된 클래스에 기능 추가하는 법
1. 만들어져 있는 클래스 가져다 사용 
2. 기존 클래스 상속
3. 기존 클래스를 활용한 재구성
4. 직접 구현

동기화 정책 문서화가기
- 구현한 클래스가 어느 수준까지 스레드 안정성을 보장하는지 기록
- 잘 정리된 문서를 통하여 유지보수에 도움이 된다

#### 5장 [5-1 ~ 5-3]
동기화된 컬렉션 클래스  
 - e.g., vector, hashtable
 - Collections.synchronizedXX 메소드를 사용해 동기화
 - 문제점
   - 아래의 예시는 thread safe 하지만 섞여 동작하거나 순서대로 동작하여 기대하지 않은 결과를 만든다
   - 동기화 하면 병렬 처리의 이점을 잃어버린다
```java
public static Object getLast(Vector list){
    synchronized(list){
        int lastIndex = list.size() -1;
        return list.get(lastIndex);
    }
}

public static void deleteLast(Vector list){
    synchronized(list){
        int lastIndex = list.size() - 1;
        list.remove(lastIndex);
    }
}
```
 - Iterator & ConcurrentModificationException
    - 반복문이 실행되는 동안 동기화 오류를 나타내는 Exception 클래스이다
    - 이를 방지하기 위하여 Loop 문을 동기화하면 deadLock 또는 starvation 발생 할 수 있다
    - Hidden Iterator 주의해야한다
      - 내부적으로 iterator 사용하여 ConcurrentModificationException 발생 가능하다
        - 예시로 StringBuilder.append, toString 등이 있다.
 ```java
public class HiddenIterator {

    @GuardedBy 
    private final Set<Integer> set = new HashSet<Integer>();

    public synchronized void add(Integer i) {
        set.add(i);
    }

    public synchronized void remove(Integer i) {
        set.remove(i);
    }

    public void addTenThings(){
        Random r = new Random();
        for(int i=0; i<10; i++) {
            add(r.nextInt());
        }
        
        System.error.println("DEBUG: added ten elements to"+ set); // iterator hidden
    }
}
```
병렬 컬렉션

블로킹 큐와 프로듀셔-컨슈머 패턴