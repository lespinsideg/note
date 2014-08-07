#java.lang.verifyerror expecting a stackmap frame at branch target
##상황
- code coverage를 체크해야 함
- 정상적으로 동작하는 서버의 서비스와 DAO를 테스트하는 JUnit Test 작성
- Test 정상 동작 확인
- Code Coverage 측정을 위해 Cobertura로 JUnit Test 구동
- 한 서비스의 메서드에서 java.lang.verifyerror expecting a stackmap frame at branch target 에러 발생

##해결책
- JVM의 -XX:UseSplitVerifier 옵션으로 stackmap frame 사용을 우회한다.

##왜?
구글링해보니 JAVA 7의 Verifier에서 발생하는 문제로 JVM에 -XX:-UseSplitVerifier 옵션으로 이전 Verifier를 사용하면 된다고 한다.
그런데 왜 이런 문제가 발생할까? 분명 컴파일 환경도 JDK 1.7 이고 실행환경도 JVM 1.7인데 자기가 컴파일한 코드를 자기가 못 돌린다? 그것도 실제 서비스나 JUnit Test에서 문제가 없던 클래스인데..

####[Java 7 Bytecode Verifier: Huge backward step for the JVM](http://chrononsystems.com/blog/java-7-design-flaw-leads-to-huge-backward-step-for-the-jvm)
좀 더 찾아보니 이런 글이 있다. chronon이라는 자바 디버깅 툴을 개발하는 회사사이트 블로그에 Prashant Deva라는 사람이 쓴 Java 7의 Verifier에 대한 깊은 빡침이 느껴지는 글이다. 글의 요지는 Java 6에서 optional로 구현되던 ByteCode의 stack map frame이 Java 7으로 넘어오면서 필수로 바뀌었고 이것 때문에 개판이 됐다는거다.

####stack map frame은 뭘까? 
Class File내 모든 메서드는 각자의 stack map을 가진다. 이 stack map은 여러개의 frame들로 이루어져 있고 각 frame들은 ByteCode 명령들과 매핑된다. stack map frame은 ByteCode 명령의 offset과 해당 명령 시점의 지역 변수와 피연산자의 타입 정보를 가진다. 이를 이용하여 ByteCode의 명령이 수행될 때마다 타입 확인을 통해 검증(Verification by type checking)을 수행한다. 자세한 내용은 [Oracle JVM 문서](http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.10.1)에 나와있다. Class File의 major version이 50 이상인 경우 (Java 6이상)에는 stack map frame을 이용하는 Type Checking 검증을 이용하고 major version이 49이하일 경우 Type 추론을 통한 검증을 수행한다. 단, major version이 50일 경우(=Java 6) Type checking에 실패하면 Type 추론을 통한 검증(Verification by type inference)을 수행한다. 이렇게 진행을 하는 이유는 Java 6에서는 stack map frame 구현이 optional이었고 그 하위 버전에는 stack map frame이 아예 없었기 때문에 이 경우에 Type 추론을 통해 검증을 진행한다.

###컴파일 시 검증을 다 하면서 ByteCode를 수행할 때 다시 검증을 수행하는 이유는?
1. 어플리케이션은 소스 코드를 다운받아서 컴파일해 실행하는 것이 아니라 이미 컴파일된 Class File을 받아서 로드하기 때문에 이 Class File이 신뢰할 수 있는 것인지 알 수 없기 때문이다.

2. 또 한가지 문제는 컴파일 시 체크는 버전 왜곡의 문제가 있다. TradingClass의 자식 클래스읜 PurchaseStockOptions라는 클래스를 성공했다고 가정했을 경우 TradingClass의 정의가 정의가 변경되어 이전에 컴파일된 바이너리 클래스와 호환이 안될 수 있기 때문이라고 한다. 이에 대한 자세한 내용은 [Java Language Definition 문서](http://docs.oracle.com/javase/specs/jls/se7/html/jls-13.html)에 나와 있다. 

**이러한 문제로 linking-time에 검증을 수행하며 이러한 검증은 interpreter의 성능을 향상시킨다고 한다. 그 이유는**
1. operand stack의 overflow나 underflow가 없음

2. 모든 지역변수가 올바르게 사용됨

3. 모든 ByteCode 명령의 인자가 올바른 타입을 사용함

**하지만 Prashant Deva는 여기에 대해서 오라클의 이런 주장은 소똥(Bullshit) 이라고 반론한다.**
1. Verifier의 경우 JVM option을 통해 통째로 끌 수 있지만 대부분의 사람들이 그렇게 이용하지 않는 이유는 성능에 영향을 거의 미치지 않기 때문이다. Verifier는 클래스가 로드되는 시점에 한번 수행되는데 이 시점에서 사용되는 시간의 대부분은 IO에 있는 것이지 verifier 수행에 있는 것이 아니기 때문이다.

2. Stack map을 이용한다고 verifier 전체가 없어지는 것이 아니라 stack map은 그저 verifier가 하는 일의 일부만 없앴기 때문이다.

3. Verifier는 하위 호환성을 위해 여전히 stack map을 이용하지 않는 검증을 지원해야 하고 이 때문에 JVM은 stack map을 이용하는 방식(Verification by type checking) 과 이용하지 않는 방식(Verification by type inference)을 모두 유지해야 한다. 이는 JVM의 복잡도를 증가시킬 뿐 아니라 JVM을 설계하는 디자이너를 괴롭히는 일이다.

4. Java 7이전에 존재하는 Type 추론 방식은 JVM에 내장된 방식으로 전문적인 컴파일러 디자이너가 작성한 효율적이고 신뢰할 수 있는 코드이다. 하지만 stack map이 사용되기 시작하면서 ByteCode를 조작하는 툴(CGLIB, ASM Framework 등)을 개발하는 사람들은 직접 stack map을 작성해야 한다. 하지만 그들은 컴파일러를 깊게 이해하는 개발자가 아닐 수 있기 때문에 비효율적이고 에러가 발생하기 쉬운 코드를 작성할 수 있다. 또한 이러한 stack map 구현에 많은 시간을 할애하여 정작 어플리케이션을 개발할 수 있는 시간이 줄어든다.

5. 오늘날 사용되는 거의 모든 어플리케이션은 ByteCode instrumentation 방식을 이용한다. Spring AOP, EclipseLink, JRebel, Chronon, YourKit, NewRelick과 같은 툴을 이용한다면 당신의 ByteCode는 컴파일 된 이후 변경될 것이다. 그렇다면 런타임에 stack map이 작성되어야 하는데 이 stack map은 4번에서 말한 컴파일러 디자이너가 아닌 개발자들에 의해서 비효율적이고 에러가 발생하기 쉬운 코드로 작성될 것이기 때문이다.

* ByteCode Instrumentation : 이미 컴파일된 클래스의 ByteCode를 JVM이 로드하여 수행하기 이전 시점에 수정하는 것. ByteCode Injection이라고도 하며 CGLIB, ASM Framework등이 이러한 것을 지원해주는 툴들이다. AOP는 이러한 방식으로 어플리케이션 단면에 코드를 삽입한다.

Prashant Deva는 stack map 도입은 결국 자바 에코시스템을 망치는 길이고 없어져야 한다고 주장한다. 그리고 stack map 이슈에 대한 해결책으로
1. 항상 -XX:-UseSplitVerifier 옵션을 이용하여 이전 verifier를 이용하도록 한다. 실제도 chronon은 이 방식으로 stack map 이슈를 해결한다고 한다.
3. 오라클이 Java 8에서 stack map을 다시 optional로 변경하도록 탄원한다. 
라고 했지만 글 마지막에 오라클은 결국 -XX:-UseSplitVerifier 사용을 deprecated 해버렸다고 나와있다. deprecated하는 이유는 WLS Group(WebLogic Server Group)이 더이상 해당 옵션을 사용하지 않기 때문이라고.. (http://bugs.java.com/view_bug.do?bug_id=8009595)

이전 Verifier 사용에 대해서는 Java SE 7 JVM 문서에도 나와있는데 stack frame 도입으로 Class File을 조작하는 툴들은 stack frame을 구현해야 하지만 구현에 시간이 필요하기 때문에 JVM에서 이전 verifier를 이용할 수 있도록 한시적으로 허용한다라고 적혀있다.들

####그렇다면 정말 stack map을 작성하는 것이 복잡한가?
여기에 대해서 Prashant Deva는 다음과 같은 예를 들었다.

```java
if(condition)
    a = new MyThis();
else
    a = new MyThat();
a.superclassMethod();
```

위와 같은 코드에서 ASM Framework는 다음 과정으로 stack frame을 구성한다.
1. 코드 마지막 줄의 a의 타입은 MyThis와 MyThat은 공통 부모 클래스 중 하나이다.
2. 하지만 runtime으로 bytecode instrumentation을 진행하는 시점에서 클래스들의 계층구조를 알 수 없다.
3. reflection을 이용해서 클래스를 동적으로 로드하여 계층구조를 조사한다. 하지만 여기서 결함이 발생한다.
	1. static class initializer가 로드되어야 한다. 하지만 이것은 프로그래머의 의도가 아니며 버그를 발생시킬 수 있다.
	2. Bytecode 수정 도중에 class가 불러졌기 때문에 해당 클래스가 로드되었을 때 ByteCode instrumentor가 호출되지 않고 그 클래스의 Bytecode를 수정할 수 없다.
	3. 부모클래스와 자식클래스가 다른 클래스로더를 사용하여 ClassNotFoundException이 발생하기 쉽다.
	4. 이것을 실제로 구현하기 위해서는 데이터 흐름 분석에 전문가가 되어야 한다.

* static class initializer가 사용되는 시점 (http://docs.oracle.com/javase/specs/jls/se7/html/jls-12.html#jls-12.4.1)

*
1. 클래스의 인스턴스가 생성될 때
2. 클래스의 static 메서드가 수행될 때
3. class의 static 필드가 할당될 때
4. 상수가 아닌 static 필드가 사용될 때
5. Top Level 클래스에 포함된 assert 구문이 수행될 때
6. Class.forName() 이 호출될 때 

본문에서 말한 static class initializer가 로드되는 경우는 6번의 경우를 말한다.

** Top Level Class는 Nested Class가 아닌 것을 말한다.

* ByteCode Instrumentor는 Java의 main() 메서드가 실행되기 이전에 premain() 메서드로 호출된다. 그렇기 때문에 이미 main() 메서드가 실행 중인 런타임 시점에서 다시 instrumentor를 호출할 수 없다.
해당 내용(http://docs.oracle.com/javase/7/docs/api/java/lang/instrument/package-summary.html)

###데이터 흐름 분석이 얼마나 복잡한가?
이에 대해서 Prashant Deva는 다음과 같이 말한다.
1. ASM Framework를 보면 알 수 있는데 ASM Framework는 데이터 흐름 분석에 1000줄 이상을 할애 하고 있고 이것이 정상적으로 동작하도록 하는동안 버전이 2나 올라갔다. 그럼에도 불구하고 이러한 '공통 부모클래스 찾기' 문제를 해결하는 방식에 결함이 있다.
2. 또한 Google에서 Stack Map Frames를 검색해보면 상위 10개 결과의 대부분이 에러 메시지에 대한 내용이고 심지어 stack map frames가 무엇인지에 대한 내용은 포함되지도 못했다.
3. ASM Framework의 '공통 부모클래스 찾기'는 제대로 동작하지 않으며 chronon은 이 구현을 변경하는게 너무 복잡하고 어렵다고 판단하여 jvm option을 이용하여 이전 verifier를 사용하는 방식을 이용한다.

##결론
서버에서 정상적으로 구동하는 서비스 클래스가 Cobertura에서만 돌아가지 않았던 이유는 Cobertura가 내부적으로 ASM Framework를 사용하여 ByteCode Instrumentation을 수행하는데 이 때 stack frame 구성에 문제가 발생한 것이다.(테스트한 나머지 클래스들은 기능이 단순하여 각 메서드의 라인수가 5줄 내외였다.) 내가 사용했단 Cobertura버전은 1.9 버전대 였고 Cobertura가 사용하는 ASM Framework의 버전은 3.2였다. CGLIB 3.0, ASM Framework 4.0 이상부터 Java 7의 Bytecode 문제를 해결했고 Cobertura는 2.0.1 이상부터 ASM Framework 4.0 보다 높은 버전을 이용한다.