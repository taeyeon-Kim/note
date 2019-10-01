## 개요
JDK8에서부터 PermGen 영역은 Metaspace 영역으로 대체되었다. PermGen과 Metaspace 영역의 차이를 간략하게 알아보자.

## PermGen
**PermGen(Permanent Generation)  main heap memory에서 분리된 special한 heap영역입니다.**

> 다른 문서를 찾아보면 heap 영역이 아니다라고 하는데 오라클이 구현한 JVM에서는 heap영역이다. [오라클 문서](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/memleaks.html)를 보면 아래와 같은 문장이 있다.
> The permanent generation is the area of the heap where class and method objects are stored. 
> 또한 [PerGem 영역도 GC의 대상](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/tooldescr.html#gblmm)이 된다.

 - JVM은 PermGen에 로드된 class metadata를 계속 찾습니다. 
 - JVM은 static content(static methods, primitive variables, references to the static objects)를 PermGen에 저장합니다.
 - PermGen에는 바이트코드, 이름 및 JIT 정보가 포함되어있다.
 - java7 이전에는 String pool도 PermGen 영역이었다. 이 와 관련된 문서는 [다음 링크](./java-string-pool.md)를 참조하자.
 - Default size는 32bit JVM은 64MB, 64bit JVM은 82MB이다.
 - 다음 옵션을 통해 default size는 변경이 가능하다.
     - `-XX:PermSize=[size]`: PermGen의 초기 또는 최소 size
     - `-XX:MaxPermSize=[size]`: PerGen의 최대 size
 - JDK8에서 완전히 사라졌다.
 - 메모리 size가 제한되어있는 PermGem은 OOM(Out Of Memory) 발생에 관여한다. 다시 말해 Class Loader들은 적절히 GC되지 않았고 이는 메모리 누수로 이어졌다.
 
 
## Metaspace
 - JDK8에서 PermGen 영역 대신 만들어진 메모리 영역이다.
 - PermGen과 가장 중요한 차이점은 메모리 할당을 처리하는 방법이다. Metaspace는 default에서 자동적으로 커진다. 
 - 메모리 tune-up을 위한 새로운 flag도 있다.
     - `MetaspaceSize and MaxMetaspaceSize`: Metaspace의 최대 size
     - `MinMetaspaceFreeRatio`: GC 후 사용 가능한 class metadate 크기의 최소 비율(GC를 유발할 class metadate에 할당된 size의 증가를 방지한다.)
     - `MaxMetaspaceFreeRatio`: GC 후 사용 가능한 class metadate 크기의 최대 비율(GC를 유발할 class metadate에 할당된 size의 감소를 방지한다.)
  - GC는 class metadata 사용량이 최대 metadata에 도달하면 죽은 class의 정리를 자동으로 트리거한다.
  - 이로 인해 OOM이 발생이 적어지게 되었다.
