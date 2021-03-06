---
title: "[java] 자바 메모리누수(with 힙덤프) 분석하기"
layout: post
category: [tech]
tags: [tech, java, mat]
date: 2019-05-29 19:00:00
---

아직까지는 메모리 릭 등의 문제로 힙덤프(HeapDump)를 생성하고 이를
분석하는 일을 해본 경험은 없다(…​ 이 넓고 얕은 지식이여).

우아한형제들 회사 블로그(도움이 될수도 있는 JVM memory leak 이야기,
<http://woowabros.github.io/tools/2019/05/24/jvm_memory_leak.html>)를
보고 난 이후, '방법은 알아(정리해)둬야겠다' 싶어 정리한다.

인터넷을 찾아보니 이미 관련하여 잘 정리된 문서들이 많다.

> **Note**
>
> 난 왜 이제야 찾아보고 있는지…​.

-   [Java Platform, Standard Edition Troubleshooting
    Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/toc.html)

> **Note**
>
> 발번역이 시작된다!!

`OnOutOfMemoryError` 는 메모리 누수(Memory leak) 상황이 발생했을 때
일어난다. 자바에서는 `java.lang.OutOfMemoryError` 예외(Exception)이
발생한다. 자바는 개체를 힙(Heap) 공간에 생성하고 이 생성위치에 대한
주소를 가지고 개체 참조(Object reference)하는 방식으로 사용한다. 개체를
생성하는 과정에서 힙 공간에 개체를 할당하기 위한 공간이 부족한 경우
발생한다. 이 경우 가비지 컬렉터는 새로운 개체를 생성할 수 있는 공간을
확보할 수 없다. 드물게 가비지 컬렉션을 수행하는데 과도한 시간이 소비되며
메모리를 사용하지 못하는 상황에서도 발생할 수 있다. 기본 할당조건을
충족하지 못하는 경우 네이티브 라이브러리 코드에 의해 발생할 수 있다(예:
스왑공간 부족).

이 때에 `java.lang.OutOfMemoryError` 예외가 발생하고
스택트레이스(StackTrace)가 출력된다.

`OutOfMemoryError` 예외 종류 및 원인
====================================

`OutOfMemoryError`를 진단하기 위해 가장 우선되는 것은 예외의 원인을
판별하는 것이다. 원인을 찾을 수 있도록 예외 출력 텍스트에는 ekdmarhk
rkxdms 세부 메시지가 포함된다.

    $ java -XX:+HeapDumpOnOutOfMemoryError -mn256m -mx512m ConsumeHeap
    java.lang.OutOfMemoryError: Java heap space
    Dumping heap to java_pid2262.hprof ...
    Heap dump file created [531535128 bytes in 14.691 secs]
    Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
            at ConsumeHeap$BigObject.(ConsumeHeap.java:22)
            at ConsumeHeap.main(ConsumeHeap.java:32)

`java.lang.OutOfMemoryError: Java heap space`(Java 힙 공간)
-----------------------------------------------------------

-   원인: 자바 힙 공간에 새로운 개체를 생성할 수 없는 경우 발생한다. 이
    오류가 반드시 메모리 누수를 의미하는 것은 아니다. 지정한 힙
    크기(혹은 기본 크기)가 애플리케이션에 충분하지 않은 경우 발생한다.
    다른 경우, 생명주기가 긴 애플리케이션의 경우

    혹은 finalize를 과도하게 사용하는 애플리케이션에서 발생하기도 한다.
    클래스에 finalize 메서드가 있는 경우 해당 개체에 대한 가비지컬렉션
    시간에 공간을 확보하지 못한다. finalizer가 finalizer 큐에 담긴
    finalizer 큐를 처리하는 속도보다 빠른 속도로 쌓이면서 힙 공간이
    가득차면서 발생할 수 있다.

-   조치: finalization 보류 상태의 객체를 모니터링하는 방법을 고려한다.

    -   JConsole 관리도구에서 finalization 상태의 객체 숫자를 확인할 수
        있다. 이 도구의 **Summary** 탭에서 메모리 분석 부분에서 확인할
        수 있다. 개수는 대략적일 수 있지만 애플리케이션에 어떤 영향을
        끼치는지 확인하고 설정을 변경할 수 있다.

    -    오라클 솔라리스와 리눅스 운영체제에서는 `jmap -finalizerinfo`
        명령어를 통해 대기상태의 finalizer 정보를 확인할 수 있다.

    -   `java.lang.management.MemoryMXBean#getObjectPendingFinalizationCount()`
        메서르를 이용해서 대략적인 숫자를 확인할 수 있다.

`ava.lang.OutOfMemoryError: GC Overhead limit exceeded`(GC 오버해드 한도 초과)
------------------------------------------------------------------------------

-   원인: 가비지 컬렉터 실행과정에서 자바 프로그램이 느려지는 경우
    발생한다. 가바지 수집 후, 자바 프로세스가 자바 컬렉션을 수행하는데
    걸리는 시간의 약 98% 이상을 소비하고 힙의 2% 미만이 복구된 상태에서
    지금까지 수행하는 과정에서 가비지 컬렉션 중
    `java.lang.OutOfMemoryError`가 5번 이상 생성되는 경우 발생한다. 이
    예외는 일반적으로 데이터를 할당하는 데 필요한 공간이 힙에 없는 경우
    발생한다.

-   조치

    -   힙 크기를 늘린다.

    -   `-XX:-UseGCOverheadLimit` 선택사항을 추가하여
        `java.lang.OutOfMemoryError` 가 발생하는 **초과 오버헤드 GC 제한
        명령**을 **해제**할 수 있다.

`java.lang.OutOfMemoryError: Requested array size exceeds VM limit`(요청 배열 크기가 VM 제한을 초과합니다.)
-----------------------------------------------------------------------------------------------------------

-   원인: 애플리케이션(혹은 애플리케이션이 사용하는 API)이 힙 공간보다
    큰 배열을 할당을 시도하는 경우 발생한다. 예를 들어, 애플리케이션이
    512MB 크기의 배열을 할당하려하지만 힙의 최대크기가 256MB인 경우
    요청배열크기가 VM 제한을 초과하면서 `java.lang.OutOfMemoryError`를
    던진다.

    -   힙 공간 사이즈가 너무 작은 경우

    -   배열 요소를 계산하고 더하는 등 배열을 키우는 경우

`java.lang.OutOfMemoryError: Metaspace`
---------------------------------------

-   원인: 자바 클래스 메타데이터(자바 클래스에 대한 VM 내부표현)는 원시
    메모리(== 메타공간)에 할당된다. 클래스 메타데이터가 할당될
    메타공간이 모두 소모되면() `java.lang.OutOfMemoryError: Metaspace`가
    발생한다. 클래스 메타데이터가 할당될 공간은 `MaxMetaSpaceSize`
    매개변수로 제한된다.

-   조치: `MaxMetaSpaceSize` 값을 늘려 설정한다. `MaxMetaSpaceSize`는
    자바 힙과 동일한 주소 공간에 할당된다. 자바 힙의 크기를 줄이면 더
    많은 공간을 확보할 수 있다. 자바 힙 공간에 여유가 있는 경우에
    고려해볼 수 있다.

`java.lang.OutOfMemoryError: request size bytes for reason. Out of swap space?`
-------------------------------------------------------------------------------

-   원인: 자바 HotSpot VM 코드가 네이티브 힙이 고갈되어 네이티브 힙에
    할당할 수 없는 경우 발생한다. 이 메시지는 실패한 요청의 (바이트)
    크기와 메모리 요청의 이유를 나타낸다. 대개의 경우 할당에 실패한 소스
    모듈의 이름을 출력한다.

-   조치: 네이티브 힙 고갈의 경우는 힙 메모리 로그 및 메모리 맵 정보를
    분석하는 것이 유용하다. 이런 유형은 운영체제의 문제 해결 유틸리티를
    사용하여 문제를 진단할 수 있다.

    -   치명적인 오류 로그 파일을 살펴보고자 한다면 [Fatal error
        log](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/felog.html#fatal_error_log_vm)를
        참조하기 바란다.

    -   운영체제 분석도구: [Native Operation System
        tools](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr020.html#BABBHHIE)

`java.lang.OutOfMemoryError: Compressed class space`(압축된 클래스 공간)
------------------------------------------------------------------------

-   원인: 64비트 플랫폼에서 클래스 메타데이터 포인터는 32비트
    오프셋(`UseCompressedOops`)으로 표현된다. 이 방식은
    `UseCompressedClassPointers`(기본값 **활성화,on**)으로 제어할 수
    있으며 활성화되면 클래스 메타데이터가 사용할 수 있는 공간의 크기가
    고정된다. `UseCompressedClassPointers`에 필요한
    `CompressedClassSpaceSize`를 초과하면
    `java.lang.OutOfMemoryError: Compressed class space`를 던진다.

-   조치: `CompressedClassSpaceSize` 크기를 키우거나
    `UseCompressedClassPointers`를 비활성화 시킨다.

    > **Note**
    >
    > `CompressedClassSpaceSize`는 허용범위가 있어서
    > `-XX:CompressedClassSpaceSize=4g`로 설정할 경우 다음과 같은
    > 메시지를 볼 것이다.
    >
    >     CompressedClassSpaceSize of 4294967296 is invalid; must be between 1048576 and 3221225472.

`java.lang.OutOfMemoryError: reason stack_trace_with_native_method`
-------------------------------------------------------------------

-   원인: 이 메시지가 출력되는 것은 원시 메서드에서부터 스택 드레이스가
    출력되었다는 것을 의미하며, 네이티브 메서드에 할당 오류가 발생했음을
    의미한다. 이 메시지가 이전 메시지들과 다른 점은 JVM 코드가 아니라
    Java Native Interface(JNI) 또는 원시메서드에서 할당실패가
    감지되었다는 것이다.

-   조치: 이 예외가 발생했다면 운영체제가 제공하는 유틸리티를 이용해서
    문제점을 진단해야 한다.

    -   운영체제 분석도구: [Native Operation System
        tools](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr020.html#BABBHHIE)

> **Note**
>
> 발번역이 끝났다!!

자바 애플리케이션에서 메모리 누수가 생기는 경우는 종종 발생한다. 이를
해결하기 위해서 메시지를 잘 살펴보고 생성된 힙덤프 파일 및 로그 파일을
살펴보면서 원인을 분석하여 문제를 찾아내는 것이 중요하다.

메모리 덤프 생성옵션 설정 및 분석(by MAT)
=========================================

자바 애플리케이션이 실행 중에 메모리 누수 등의 문제가 발생했을 때 관련된
문제를 정리하여 덤프파일을 생성할 수 있다.

상황이 발생했을 때 힙덤프를 생성하는
선택사항(`-XX:+HeapDumpOnOutOfMemoryError`)과 위치를 지정하는
선택사항(\`-XX:HeapDumpPath=/var/log \`)를 추가한다. 이렇게 설정해놓으면
`OutOfMemoryError`가 발생했을 때 `java_pid{pid}.hprof`파일이 지정된
위치에 생성된다.

    -Xms2g -Xmx4g -XX:+UseG1GC -XX:NewRatio=7 -XX:+PrintGCDetails -XX:+PrintGCDateStamps -verbose:gc -Xloggc:/var/log/gc.log -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log

> **Note**
>
> 생성되는 파일의 크기가 700메가가 넘는다!!

해당 파일을 열어볼 수 있는 몇 가지 도구가 있는데 가장 널리 알려져 있는
것이 MAT(Memory Analyzer, <https://www.eclipse.org/mat/>)다. MAT는
**자바 힙 분석**을 위한 풍부한 기능을 제공하여 메모리 누수 및 메모리
소비 감축요소를 찾을 수 있도록 돕는다.

MAT에서 `java_pid17468.hprof`를 열어보면 다음과 같은 다양한 파일이
생성된다. 의외로 읽어내는 속도는 빠르다.

    java_pid17468.hprof
    java_pid17468.a2s.index
    java_pid17468.domIn.index
    java_pid17468.domOut.index
    java_pid17468.i2sv2.index
    java_pid17468.idx.index
    java_pid17468.inbound.index
    java_pid17468.index
    java_pid17468.o2c.index
    java_pid17468.o2hprof.index
    java_pid17468.o2ret.index
    java_pid17468.outbound.index
    java_pid17468.threads
    java_pid17468_Leak_Suspects.zip

![메모리 누수 발생지점을 추축한다.]({{"/assets/post/2019-05-29/2019-05-29-01.png" | absolute_url }})

![MAT Suspect 화면]({{"/assets/post/2019-05-29/2019-05-29-02.png" | absolute_url }})

이런 추측 지점을 잘 살펴보고 개선할 수 있는 방법을 모색하여 코드를
조금씩 개선해나가면 좀 더 안정적인 시스템을 만들어갈 수 있을 것이다.

VisualVM
========

MAT외에도 가볍게 살펴볼 수 있는
[**VisualVM**](https://visualvm.github.io/)도 있다. VisualVM은 [JDK에
포함된
버전](https://docs.oracle.com/javase/7/docs/technotes/guides/visualvm/index.html)과
[깃헙에서 제공하는 버전](https://visualvm.github.io/), 2가지 버전이
존재한다. Oracle JDK 9부터 VisualVM은 GraalVM에 대한 분석도 가능해졌다.
GraalVM에서 실행되는 애플리케이션을 분석하는 용도로 유용한 듯 싶다.

![VisualVM main]({{"/assets/post/2019-05-29/2019-05-29-03.png" | absolute_url }})

MAT처럼 친절하지는 않지만 제공되는 정보들을
통해 메모리 누수가 발생하는 지점을 유추해볼 수 있다.

![VisualVM 예제화면]({{"/assets/post/2019-05-29/2019-05-29-04.png" | absolute_url }})


정리
====

꽤 오랫동안 개발자로 지내왔지만, 애플리케이션에서 문제가 발생했을 때
이를 해결하는 과정(트러블슈팅, TroubleShooting)에는 크게 관여하지
못했다(자랑이다…​). 이런저런 이유가 있겠지만 어찌되었든 해야할 상황에
직면했을 때 무엇을 할 수 있을지를 고민하는 시간을 줄일 수 있도록 관련
내용을 정리해둔다.

최근에 회사블로그에 올라왔던 ['도움이 될수도 있는 JVM memory leak
이야기'](http://woowabros.github.io/tools/2019/05/24/jvm_memory_leak.html)는
좋은 계기가 되었다.

> **Important**
>
> 자신감있게 문제를 일으키자!!

참고문헌
========

-   [Java Platform, Standard Edition Troubleshooting
    Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/toc.html)

-   [Understand the OutOfMemoryError
    Exception](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/memleaks002.html)

-   [도움이 될수도 있는 JVM memory leak
    이야기](http://woowabros.github.io/tools/2019/05/24/jvm_memory_leak.html)

-   [Command-Line Options -
    오라클](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/clopts.html)

-   [Java Memory Analysis](http://kwonnam.pe.kr/wiki/java/memory)
