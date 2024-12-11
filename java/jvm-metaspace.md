## JVM 동작 방식

- .java 파일을 .class로 컴파일
- Class Loader는 .class 파일을 JVM Runtime Data Area로 동적 로딩
- Execution Engine은 Runtime Data Area에 로딩된 .class 해석 및 실행 (이 과정에서 GC 작동과 스레드의 동기화 시작)

> JIT(Just-In-Time) 컴파일러는 자주 호출되는 코드 블록을 기계어로 변환하고 캐싱해 성능을 향상시킵니다. JIT 컴파일러를 통해 최적화된 바이트 코드는 Execution Engine에 의해 실행됩니다.

<br>

## Metaspace (Java 8 이후)

![](/java/img/jvm-metaspace.png)

> Move part of the contents of the permanent generation in Hotspot to the Java heap and the remainder to native memory...The proposed implementation will allocate class meta-data in native memory and move interned Strings and class statics to the Java heap. https://openjdk.org/jeps/122

- java.lang.OutOfMemoryError: PermGen space로 인해 Java 8 버전부터 Permanent Generation(PermGen) 제거 
- PermGen에 있던 Metadata, Runtime Constant Pool이 Native memory 영역의 Metaspace로 이동

<br>

> Heap 영역에 객체를 생성하면 객체에 대한 Lock, Class 정보가 담긴 Object Header가 추가됩니다. Class 정보가 담긴 Object Header에는 가상 메서드 테이블(vtable) 주소가 있는데 이 vtable을 통해 Metaspace에서 정보를 가져오기 때문에 동적바인딩(@Override)이 가능합니다.

<br>


### GC
- PermGen 영역은 GC 대상이 아니므로 static 변수가 거의 제거되지 않았지만, Java8 이후 Heap 영역으로 이동하여 제거 대상이 됨
- Native memory는 OS가 관리하는 메모리 영역이지만, GC의 영향을 받을 수 있음 https://stackoverflow.com/questions/24074164/what-is-the-use-of-metaspace-in-java-8

> 참고로 현재 default GC는 G1 GC이며, Java 21 버전에서는 ZGC young, old 객체에 대한 별도의 Generation으로 분리해 성능을 향상시켰다고 합니다. https://openjdk.org/jeps/439

<br>

### 🤔 궁금한 점

- Metaspace과 별개로 Method Area 개념이 아직도 존재한다고 하는데, Method Area에서는 무엇이 관리되는지