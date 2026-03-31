### 답변

>Java의 GC는 JVM이 Heap 영역에서 더 이상 참조되지 않는 객체를 자동으로 제거하는 메모리 관리 기능입니다.  
GC는 Reachability 기반으로 동작하며, GC Root에서 도달할 수 없는 객체를 수거합니다.  
또한 Java는 Generational GC 방식을 사용해 Heap을 Young 영역과 Old 영역으로 나누고, 대부분의 객체가 빨리 생성되고 빨리 사라진다는 가정을 활용해 효율적으로 메모리를 관리합니다.  
다만 GC 수행 시 Stop-The-World가 발생해 일시적으로 애플리케이션이 멈출 수 있다는 단점이 있습니다.

---

Java의 GC는 **Heap 영역의 객체를 자동으로 회수하는 메모리 관리 메커니즘**입니다.  
개발자가 직접 `free`를 호출하지 않아도,  
더 이상 참조되지 않는 객체를 JVM이 탐지해 제거합니다.

기본 원리는 **Reachability(도달 가능성) 분석**입니다.  
GC Root(스택의 지역 변수, static 변수 등)에서 시작해  
참조 그래프를 따라가며 도달 가능한 객체는 유지하고,  
도달 불가능한 객체를 수거합니다.

Java GC는 대부분 **Generational GC** 전략을 사용합니다.

- Young Generation: 새로 생성된 객체가 위치  
    → Minor GC 발생
    
- Old Generation: 오래 살아남은 객체  
    → Major 또는 Full GC 발생
    

이는 대부분 객체가 금방 사라진다는  
“약한 세대 가설(Weak Generational Hypothesis)”에 기반합니다.

GC 방식에는 여러 알고리즘이 있습니다.

- Mark-Sweep: 표시 후 제거, 단편화 발생 가능
    
- Mark-Compact: 압축하여 단편화 방지
    
- Copying: 한 영역에서 다른 영역으로 복사
    
- G1, ZGC 등: 지연 시간을 최소화하기 위한 현대적 GC
    

GC의 장점은  
메모리 관리 실수를 줄이고 개발 생산성을 높인다는 점입니다.

단점은  
GC 시점에 Stop-The-World가 발생할 수 있고,  
예측하기 어려운 지연이 생길 수 있다는 점입니다.

정리하면,  
Java GC는 도달 가능성 분석과 세대별 관리 전략을 기반으로  
자동 메모리 회수를 수행하는 JVM의 핵심 메커니즘입니다.