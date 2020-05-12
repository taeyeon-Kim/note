## 개요
`String` Object는 자바에서 가장 많이 사용되는 class입니다.<br/>
JVM에 의해 String들이 저장되는 특별한 메모리 영역인 **Java String Pool**에 대해 알아봅시다.

## String Interning
자바에서는 String들의 불변성 덕분에 JVM은 **풀에 각 literal String의 복사본 하나만 저장**하여 메모리를 최적화합니다.<br/>
이 과정을 Interning이라 합니다.<br/>
만약 literal String을 생성한다고 해보자.
 1. JVM은 String pool에서 value가 같은 값을 찾습니다.<br/>
 2.
    1. 만약 찾는다면, Java 컴파일러는 메모리를 추가로 할당하지 않고 메모리 주소에 대한 reference를 리턴합니다.
    2. 만약 찾지 못한다면, interned하고 reference를 리턴합니다.
  
## 생성자를 사용한 String 할당.
`new` 연산자를 통해 String을 만들면, Java 컴파일러는 객체를 생성하고 JVM에 의해 예약된 heap 영역에 저장합니다.<br/>
이렇게 생성되는 String은 모두 자신만의 memory 영역과 주소를 가집니다.

## String Literal vs String Object
위 두 내용을 보면 `String literal`을 사용는 것이 좋습니다. 이를 통해 더 읽고 쉽고 컴팡일러가 코드 최적화를 할 수 있게합니다.

## Manual Interning
`intern()` 메서드를 통해 수동으로 interning할 수 있습니다. String을 String pool에 저장하고, JVM은 해당 String이 필요할 때 reference를 return합니다.
```java
String constantString = "interned Baeldung";
String newString = new String("interned Baeldung");
 
assertThat(constantString).isNotSameAs(newString);
 
String internedString = newString.intern();
 
assertThat(constantString).isSameAs(internedString);
```

## Garbage Collection

#### java 7 이전
JVM은 **PermGen**영역에 고정된 size로 java String pool을 위치했습니다. java String pool은 런타임시에 size를 늘릴 수 없으며, GC의 대상이 아니었습니다.<br/>
이는 만약 너무 많은 `String`을 intern하면, OutOfMemory error의 위험이 있습니다.

#### java 7 이후
java String pool은 **Heap**영역에 위치하여 JVM에 의해 garbage collected될 수 있게되었습니다.<br/>
이는 참조되지 않은 String은 메모리에서 해제되기 때문에 OutOfMemory error의 위험을 줄게되었습니다.

## 성능 및 최적화

#### java 6
jvm의 **MaxPermSize** 옵션을 통해 PermGen 크기를 늘려 성능 최적화를 했습니다.
```
-XX:MaxPermSize=1G
```

#### java 7
java string pool size를 확인하는 2가지 옵션을 통해 string pool을 최적화 할 수 있습니다.
```
-XX:+PrintFlagsFinal
```
```
-XX:+PrintStringTableStatistics
```
만약 pool size(buckets)를 늘리고 싶다면, `StringTableSize` JVM 옵션을 사용하면 됩니다.
```
-XX:StringTableSize=4901
```
java `7u40`이전에는 1009 buckets가 default size이며 `7u40`에서 java 11까지는 60013이며 현재는 65536이 default입니다.<br/>
**pool size를 늘리면 메모리를 더 많이 소비하지만 table 안에 String을 삽입에 필요한 시간을 줄일 수 있는 장점이 있다**
 

## Java 9에서 주목할 점.
Java 8까지 Strings는 내부적으로 문자 배열(char[])로 표시되었고, UTF-16으로 인코딩되어 모든 문자가 2바이트의 메모리를 사용했습니다.<br/>
Java 9에서는 **Compact Strings**이라 불리는 새로운 포맷이 추가되었습니다. 이 새로운 포맷은 저장된 내용에 따라 char[]와 byte[] 중 적절한 인코딩을 합니다.<br/>
Compact Strings 이후로 필요한 경우에만 UTF-16으로 인코딩하기 때문에, heap 메모리의 양은 현저히 떨어지게되고, GC의 overhead가 줄어들게 됩니다.

-----
[출처](https://www.baeldung.com/java-string-pool)
