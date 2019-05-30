## Class Loader란?
 Class Loader는 **런타임 동안 JVM(Java Virtual Machine)에 Java Class를 동적으로 Load해주며** JRE(Java Runtime Environment)의 일부분이다.<br/>
 JVM은 Class Loader로 인해 Java Application을 실행하기 위해 기본 파일, 파일 시스템에 대해 알 필요가 없다.<br/>
 Java Class는 한 번에 메모리에 로드되지 않고 Application이 필요할 때 로딩된다. 여기서 Class Loader가 class를 메모리에 올리는 역할을 한다.<br/> 
 
-----

## 내장된 Class Loader들
```java
class ClassLoader {
    void printClassLoaders() throws ClassNotFoundException {
        System.out.println("Classloader of ArrayList:" + ArrayList.class.getClassLoader());
        
        System.out.println("Classloader of Logging:" + Logging.class.getClassLoader());
        
        System.out.println("Classloader of this class:" + ClassLoader.class.getClassLoader());

    }
}
```
```cmd
Classloader of ArrayList:null
Classloader of Logging:sun.misc.Launcher$ExtClassLoader@677327b6
Classloader of this class:sun.misc.Launcher$AppClassLoader@18b4aac2
```
내장된 Class Loader는 **Bootstrap Class Loader(null로 표시)**, **Extension Class Loader**, **Application(System) Class Loader** 3종류가 있다.
### Bootstrap Class Loader
 - 새로운 JVM이 시작 될 때 Bootstrap Class Loader는 java.lang 패키지(그 중 Java classes들은 `java.lang.ClassLoader`의 instance에 의해 load됩니다)의 주요 java class, memory first한 runtime classes를 load합니다.
 - 쉽게 말해 JDK 내부 클래스(일반적으로 rt.jar 및 $ JAVA_HOME / jre / lib 디렉토리에있는 기타 핵심 라이브러리)를 load합니다.
 - Bootstrap Class Loader는 모든 ClassLoader의 parent입니다.
 - **Bootstrap Class Loader는 JVM의 핵심 부분이며, Native Code로 작성되어 있습니다.**(null로 print되는 이유)
 - Native Code로 작성되었기 때문에 플랫폼마다 class loader의 구현이 다를 수 있습니다.
### Extension(Platform) Class Loader
 - standard core Java classes를 확장한 class를 load합니다.
 - Bootstrap Class Loader의 child입니다.
 - 플랫폼에서 돌아가는 application이 사용할 수 있는 standard core Java classes를 확장한 class를 관리합니다.
 - `$JAVA_HOME/lib/ext` 또는  `java.ext.dirs system property`에 정의된 JDK extensions 폴더에서 load합니다. 
 - `sun.misc.Launcher$ExtClassLoader`에 구현되어 있습니다.
### Application(System) Class Loader
 - classpath에 있는 우리가 작성한 file들을 load합니다.
 - 모든 애플리케이션 수준 클래스를 JVM에 로딩하는 것을 관리합니다.
 - **`-classpath` 또는 `-cp` 옵션을 이용한 classpath 환경 변수에서 file을 찾는다.**
 - Extensions classloader의 child입니다.
 - `sun.misc.Launcher$AppClassLoader`에 구현되어 있습니다.
 ----
## Class Loader의 동작 방법
Class Loader는 JRE(Java Runtime Environment)의 일부분이다.<br/>
JVM이 클래스를 request할 때, Class Loader는 정규화된 클래스 이름(e.g. `java.nio.file.Path`)을 사용하여 클래스를 찾고 클래스 정의를 런타임에 로드하려고 시도합니다.</br>

1. `java.lang.ClassLoader.loadClass()`메서드에서 런타임에 클래스를 정의하고 load하는 역할을 합니다.
2. 만약 클래스가 아직 load되지 않았다면,  부모 Class Loader에 request를 위임합니다. 이 프로세스는 재귀적으로 실행됩니다.
3. 결국, 부모 Class Loader에서 request한 class를 찾지 못했다면, 자식 ClassLoader는 `java.net.URLClassLoader.finClass()` 메서드를 호출하여 file system 자체에서 class를 찾습니다.
4. 만약 자식 Class Loader에서도 클래스를 찾을 수 없다면, `java.lang.NoClassDefFoundError`또는 `java.lang.ClassNotFoundException.`을 throw합니다.

아래는 openjdk 8u191의 java.lang.ClassLoader.loadClass() 코드입니다.
```java
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

### Class Loader의 3가지 주요한 요소
#### Delegation Model
클래스 또는 리소스를 요청할 때, ClassLoader instance는 요청한 클래스 또는 리소스를 부모 Class Loader에 위임한다.<br/>
만약 Application Class를 JVM에 로드하려는 request가 있다고 해보자. Application Class Loader는 Extension class Loder에서 위임하고 Extension Class Loader는 Bootstrap Class Loader에 위임한다. 여기서 Extension, Bootstrap Class Loader에서 클래스를 못찾는 경우에만 Application Class Loader는 자신의 file system에서 class를 찾으려고 할 것이다.

#### Unique Classes
자식 Class Loader는 부모 Class Loader가 로딩한 클래스를 다시 로딩하지 않게 해서 로딩된 클래스의 유일성을 보장하는 것이다.<br/>
유일성을 식별하는 기준은 클래스의 **binary name**이다.

#### Visibility
자식 Class Loader는 부모 Class Loader가 로드한 클래스를 볼 수 있고 부모 Class Loader는 자식 Class Loader가 로드한 클래스를 볼 수 없다.<br/>
만약에 개발자가 만든 클래스를 로딩하는 Application Class Loader가 Bootstrap Class Loader에 의해 로딩된 `String.class`를 볼 수 없다면 애플리케이션은 `String.class`를 사용할 수 없을 것이다. 따라서 하위에서는 상위를 볼 수 있어야 애플리케이션이 제대로 동작할 수 있다.

----

## Custom ClassLoader
내장된 Class Loader만 사용해도 대부분의 경우에도 충분할 것 이다.
그러나 로컬 하드 드라이브나 네트워크에서 클래스를 로드해야 하는 경우에서는 custom ClassLoader를 사용해야 할 수 있다.

### Custom Class Loaders Use-Cases
1. 기존 바이트 코드를 수정하는 경우
2. user의 요구에 따라 클래스를 동적으로 로딩해야하는 경우
    - 예를 들어, JDBC에서는 동적인 클래스 로딩을 통해 다른 드라이버 구현간에 switching이 이루어진다.
3. 이름 및 패키지가 동일한 클래스에 대해 서로 다른 바이트 코드를 로드하는 동안 class versioning mechanism을 구현
    - 이 경우는 URL class loader(load jars via URLs)로도 해결 할 수 있다.

### Custom Class Loader 예제
```java
class CustomClassLoader extends ClassLoader {

    @Override
    public Class findClass(String name) throws ClassNotFoundException {
        byte[] b = loadClassFromFile(name);
        return defineClass(name, b, 0, b.length);
    }

    private byte[] loadClassFromFile(String fileName) {
        InputStream inputStream = getClass().getClassLoader().getResourceAsStream(
                fileName.replace('.', File.separatorChar) + ".class");
        byte[] buffer;
        ByteArrayOutputStream byteStream = new ByteArrayOutputStream();
        int nextValue = 0;
        try {
            if (inputStream != null) {
                while ((nextValue = inputStream.read()) != -1) {
                    byteStream.write(nextValue);
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        buffer = byteStream.toByteArray();
        return buffer;
    }
}
```
-----

## java.lang.ClassLoader
java.lang.ClassLoader의 essential 메서드들을 알아보자.
### loadClass()
loadClass()의 시그니처는 아래와 같다.
```java
public Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
```
 - name 파라미터: fully qualified한 클래스 name이다.
 - resolve 파라미터:
      - True: request한 class로 link한다.
      - False: 클래스가 존재하는지 여부만 결정하면 되는 경우.

Class Loader의 entry point이며 기본 구현은 아래와 같다.
1. `findLoadedClass(name)`으로 이미 load되어 있는지 확인한다.
2. 없다면 부모 Class Loader의 `loadClass(name, false)` 호출
3. 부모에서 없다면 findClass(name)으로 class를 찾는다.
```java
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

### defineClass()
defineClass()의 시그니처는 아래와 같다.
```java
protected final Class<?> defineClass(String name, byte[] b, int off, int len) throws ClassFormatError
```
이 메서드는 바이트의 배열을 클래스의 인스턴스로 변환하는 역할을 한다. 그리고 클래스를 사용하기 전에, resolve해야한다.
데이터에 유효한 클래스가 없으면 `ClassFormatError`를 throw한다.
final이라 override불가능하다.

## getParent()
getParent()의 시그니처는 아래와 같다.
```java
public final ClassLoader getParent()
```
위임이 되는 부모 Class Loader를 return한다.<br/>
null이 리턴되면 BootstrapClassLoader다.

## getResource()
getResource()의 시그니처는 아래와 같다.
```java
public URL getResource(String name)
```
name 파라미터를 이용해 리소스를 찾는다.(구분자 '/')<br/>
먼저 부모 Class Loader에 위임한다. 만약 부모가 null이면,  virtual machine에 내장된 class Loader의 path에서 찾는다.<br/>
만약 이 작업이 실패하면, findResource(name)을 실행하여 리소스를 찾는다. 리소스의 이름은 상대 path이거나 절대 path입니다.<br/>
리소스를 읽을 URL 객체를 리턴하거나, 리소스를 찾을 수 없거나 리소스를 읽을 권한이 없는 경우 null이 리턴됩니다.<br/>
중요한 점은 classpath에서 자원을 로드하며 어느 환경에서든 location-independent합니다.

----

## Context Classloaders
일반적으로 Context Classloader는 J2SE에 도입된 class-loading delegation scheme에 대한 대체 방법입니다.<br/>
앞에서 우리는 JVM의 Class Loader는 모든 Class Loader가 Bootstrap Class Loader를 제외하고 하나의 상위권을 갖는 계층적 모델을 따른다는 것을 학습했습니다.<br/>
그러나 JVM 코어 클래스가 애플리케이션 개발자가 제공하는 클래스 또는 리소스를 동적으로 로드해야 하는 경우 문제가 발생할 수 있습니다.<br/>
예를 들어, JNDI(Java Naming and Directory Interface)에서 핵심 기능은 rt.jar의 부트스트랩 클래스에 의해 구현됩니다.
그러나 이러한 JNDI가 provider가 구현한 특정 vendor(application classpath에 배포됨)를 통해 JNDI를 제공할 수 있습니다.
이러한 경우 Bootstrap Class Loader(부모)가 Application Class Loader(자식)을 visible할 수 있어야합니다.
이는 class-loading delegation scheme에서는 불가능하며 Context ClassLoader를 사용하여 구현해야합니다.<br/>
`java.lang.Thread.getContextClassLoader()`를 호출하면 특정 thread의 ContextClassLoader를 가져옵니다.<br/>
ContextClassLoader의 thread creator가 리소스 및 클래스 로드를 제공합니다.<br/>
default는 부모 thread의 class loader context입니다.<br/>