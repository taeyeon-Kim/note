많은 Java 개발자들이 JRE(Java Runtime Environment)에 의해 bytecode가 실행된다고 압니다.<br />
그러나 바이트코드를 분석하고 코드를 해석하여 실행하는 JVM(Java Virtual Machine)의 구현과 라이브러리의 조합이 JRE라는 사실은 많이 모릅니다.<br />
[JVM 스팩](https://docs.oracle.com/javase/specs/jvms/se13/html/)을 구현한 JVM 밴더(오라클, 아마존, Azul)마다 다를 수 있습니다.
 > 참고: 오라클은 11버전부터 jre를 제공안합니다. 무조건 jdk


## JVM Architecture Diagram
![JVM-architecture](./image/JVM-architecture.png)

JVM은 3가지로 나뉜다.

 [1. Class Loader SubSystem](#class-loader-subsystem)<br />
 [2. Runtime Data Area](#runtime-data-area) <br />
 [3. Execution Engine](#execution-engine) <br />
 

### Class Loader SubSystem
 - 자바의 동적 클래스 로딩 기능은 Class Loader SubSystem에 의해 동작됩니다. 런타임시에 클래스 파일을 처음 참조할 때 클래스 파일을 Loading -> Linking -> Initialization합니다. 
####  Loading
BootStrap ClassLoader, Extension ClassLoader, Application ClassLoader에 의해 class들이 load됩니다.<br/>
classLoader들은 클래스 파일을 로드하는 동안 Delegation Hierarchy Algorithm을 따릅니다.
  1. BootStrap ClassLoader
     - 새로운 JVM이 시작 될 때 Bootstrap Class Loader는 java.lang 패키지(그 중 Java classes들은 `java.lang.ClassLoader`의 instance에 의해 load됩니다)의 주요 java class, memory first한 runtime classes를 load합니다.
     - 쉽게 말해 JDK 내부 클래스(일반적으로 rt.jar 및 $ JAVA_HOME / jre / lib 디렉토리에있는 기타 핵심 라이브러리)를 load합니다.
     - Bootstrap Class Loader는 모든 ClassLoader의 parent입니다.
     - **Bootstrap Class Loader는 JVM의 핵심 부분이며, Native Code로 작성되어 있습니다.**
     - Native Code로 작성되었기 때문에 플랫폼(JVM 밴더)마다 class loader의 구현이 다를 수 있습니다.
  2. Extension ClassLoader
     - standard core Java classes를 확장한 class를 load합니다.
     - Bootstrap Class Loader의 child입니다.
     - 플랫폼에서 돌아가는 application이 사용할 수 있는 standard core Java classes를 확장한 class를 관리합니다.
     - `$JAVA_HOME/lib/ext` 또는  `java.ext.dirs system property`에 정의된 JDK extensions 폴더에서 load합니다. 
     - `sun.misc.Launcher$ExtClassLoader`에 구현되어 있습니다.
  3. Application ClassLoader
      - classpath에 있는 우리가 작성한 file들을 load합니다.
      - 모든 애플리케이션 수준 클래스를 JVM에 로딩하는 것을 관리합니다.
      - **`-classpath` 또는 `-cp` 옵션을 이용한 classpath 환경 변수에서 file을 찾습니다.**
      - Extensions classloader의 child입니다.
      - `sun.misc.Launcher$AppClassLoader`에 구현되어 있습니다.
 > 참고: [Class Loader](./class-loader-in-java.md)
 
#### Linking
레퍼런스를 연결하는 과정입니다.
  1. Verify
    - bytecode가 적절한지 검증합니다. 적절하지 않다면 error가 발생합니다.
  2. Prepare 
    - 모든 static 변수가 메모리에 할당 되고 default values값이 할당 됩니다.
  3. Resolve 
    - Method Area의 모든 symbolic memory references들이 original references으로 대체 됩니다.

#### Initialization
ClassLoading의 마지막 단계로써 모든 static 변수가 원래 값으로 할당되고 static block이 실행됩니다.
### Runtime Data Area
 1. Method Area
    - 모든 class-level 데이터를 저장합니다.
        - class-level 데이터: un-time constant pool, field, method data, 메서드와 생성자를 위한 코드
    - JVM 마다 하나만 제공하며 공유 자원입니다.
    - 여러 스레드의 메모리를 공유하므로 저장된 데이터는 thread-safe하지 않습니다.
    - Method Area의 메모리에 할당 요청이 안된다면 OOM이 발생합니다.
 2. Heap Area
    - 모든 객체와 해당 인스턴스 변수 및 배열이 여기에 저장됩니다.
    - JVM 마다 하나입니다.
    - 여러 스레드의 메모리를 공유하므로 저장된 데이터는 thread-safe하지 않습니다.저장합니다.
 3. Stack Area
    - 모든 스레드에 대해 별로의 런타임 스택이 생성됩니다.
    - 모든 메서드 call에 대해 **Stack Frame**이라는 스택 메모리에 하나의 entry로 작성됩니다.
        - Stack Frame ?
             - Local Variable Array: 얼마나 많은 지역 변수와 해당 값이 관련되어 있는지가 저장된다.
             - Operand stack: 만약 어떤 중간 연산이 필요한 경우에 Operand stack은 해당 연산을 수행하기 위한 runtime workspace로 작용한다.
             - Frame data: 메서드의 모든 symbol들이 여기에 저장됩니다. 어떤 **exception**안에서든, catch 문에서 frame data는 유지됩니다.
    - 모든 지역 변수는 스택 메모리에 생성됩니다.
    - Stack Area는 공유 자원이 아니므로 thread-safe합니다.
 4. PC Registers
    - 각 스레드에는 별도의 PC 레지스터가 있으며, 명령이 실행되면 PC 레지스터가 다음 명령으로 업데이트됩니다. 즉, 현재 수행 중인 명령의 주소를 갖습니다.
 5. Native Method Stacks
    - Native Method Stack에는 네이티브 메서드 정보가 들어 있다. 모든 스레드에 대해 별도의 네이티브 메서드 스택이 생성됩니다.
### Execution Engine
Runtime Data Area에 할당된 바이트 코드는 Execution Engine에 의해 실행됩니다. Execution Engine은 바이트코드를 읽고 그것을 하나씩 실행합니다.
  1. Interpreter 
     - Interpreter는 bytecode를 빠르게 해석하지만 실행은 느립니다.
     - Interpreter의 하나의 method를 여러번 부르면 매번 새로운 interpretation이 필요하다는 단점이 있습니다.
  2. JIT Compiler
     - JIT Compiler는 Interpreter의 단점을 해결해줍니다.
     - Execution Engine은 바이트 코드를 변환하는 데 Interpreter의 도움을 받지만, 반복된 코드를 발견하면 JIT 컴파일러를 사용하여 전체 바이트코드를 컴파일하여 네이티브 코드로 변경합니다.
     - 이 네이티브 코드는 반복되는 메서드 호출에 직접적으로 사용되어 시스템의 성능을 향상 시킨다. 
        1. Intermediate Code Generator - 중간 코드를 생성합니다.
        2. Code Optimizer - 생성된 중간 코드를 최적화합니다.
        3. Target Code Generator - 기계 코드 또는 네이티브 코드를 생성합니다.
        4. Profiler - 핫스팟(메서드를 여러 번 호출하는지 여부)을 찾는 역할을 하는 특수 component
  3. Garbage Collector
    -  unreferenced 객체들을 Collects하고 Removes한다. 

### Java Native Interface (JNI)
Native Method Libraries와 상호 작용하며 Execution Engine에 필요한 Native Libraries를 제공한다.

### Native Method Libraries
Execution Engine에 필요한 네이티브 라이브러리의 모음입니다
