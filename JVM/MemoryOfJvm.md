## 자바 메모리 영역과 메모리 오버플로우

### 📝런타임 데이터 영역
- JVM은 자바 프로그램을 실행하는 동안 필요한 메모리를 몇 개의 데이터 영역으로 나누어 관리
- 이 영역들은 각각 목적과 생성/삭제 시점이 있다.

**💡<<자바 가상 머신 명세>>에 따르면 JVM이 관리하는 메모리는 아래와 같은 런타임 데이터 영역들로 구성된다.**
<table>
    <thead>
        <tr>
            <td colspan="3" style="text-align:center; background-color:#c8d9d0; color:black; border-color:white;">
                <b>런타임 데이터 영역</b>
            </td>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td colspan="1" style="text-align:center; border-color:white; background:#4688a5; color:white">메서드 영역 [런타임 상수 풀]</td>
            <td colspan="1" style="text-align:center; border-color:white;">가상 머신 스택</td>
            <td colspan="1" style="text-align:center; border-color:white;">네이티브 메서드 스택</td>
        </tr>
        <tr>
            <td colspan="1" style="text-align:center; border-color:white; background:#4688a5; color:white">힙</td>
            <td colspan="2" style="text-align:center; border-color:white;">프로그램 카운터 레지스터( <i>하드웨어랑 헷갈리지 말것</i> )</td>        
        </tr>
        <tr> <td colspan="3"></td> </tr>
        <tr> <td colspan="3"></td> </tr>
        <tr>
            <td colspan="1" style="text-align:center; border-color:white;"> 실행 엔진 </td>    
            <td colspan="1" style="text-align:center; border-color:white;"> 네이티브 라이브러리 인터페이스 </td>    
            <td colspan="1" style="text-align:center; border-color:white;"> 로컬 메서드 라이브러리</td>    
        </tr>
    </tbody>
</table>

- 모든 스레드가 공유하는 데이터 영역
    <div style="width:20px; height:20px; border:solid 1px; border-color:black; background:#4688a5;"></div> 
- 스레드별 데이터 영역(스레드 프라이빗 *= Thread Local?)
    <div style="width:20px; height:20px; border:solid 1px; border-color:black; background:transparent;"></div> 

**💡프로그램 카운터**
- 작은 메모리 영역으로, 현재 실행 중인 스레드의 `바이트코드 줄 번호 표시기`라고 생각하면 쉽다.
- JVM 개념 모형에서 바이트코드 인터프리터는 이 카운터의 값을 바꾸어 다음에 실행할 바이트코드 명령어를 선택하는 식으로 동작
- 프로그램의 제어 흐름, 분기, 순환, 점프 등을 표현하는 것, 예외 처리나 스레드 복원 같은 모든 기본 기능이 바로 이 표시기를 활용하여 이루어진다.

```text
JVM에서 멀티스레딩은 CPU 코어를 여러 스레드가 교대로 사용하는 방식으로 구현되기 때문에 특정 시각에 각 코어는 한 스레드의 명령어만 실행하게 된다.
따라서 스레드 전환 이후 이전에 실행하다가 멈춘 지점을 정화하게 복원하려면 스레드 각각에 고유한 프로그램 카운터가 필요하다.
따라서 각 스레드의 프로그램 카운터는 서로 영향을 주지 않는 독립된 영역에 저장된다. 이 메모리 영역을 프라이빗 메모리라고 한다.  
```
- 스레드가 네이티브 메서드를 실행 중일 때 프로그램 카운터의 값은 Undefined다.

**💡자바 가상 머신 스택**
- 프로그램 카운터처럼 자바 가상 머신 스택도 `스레드 프라이빗`하며, 연결된 스레드와 운명을 같이 한다. (=생명주기가 일치한다.)
- 가상 머신 스택은 자바 메서드를 실행하는 스레드의 메모리 모델을 설명하여 준다.
- 각 메서드가 호출 될 때마다 자바 가상 머신은 스택 프레임을 만들어 `지역 변수 테이블, 피연산자 스택, 동적 링크, 메서드 반환값` 등의 정보를 저장, 그런 다음 스택 프레임을 가상 머신 스택에 푸시(push)하고, 메서드가 끝나면 팝(pop)하는 일을 반복한다.
- `스택`이라고 하면 보통 `자바 가상 머신 스택`을 가르키는데, 그중 특히 `지역 변수 테이블`을 가리킬 때가 많다.
- `지역 변수 테이블`에는 JVM이 컴파일타임에 알 수 있는 `다양한 기본 데이터 타입, 객체 참조, 반환 주소 타입`을 저장한다. 이 데이터 타입들을 저장하는 공간을 지역 변수 슬롯이라 한다.
- 일반적으로 슬롯 하나의 크기는 32비트다. 따라서 double 타입처럼 길이가 64비트인 데이터는 슬롯 두개를 차지하며, 나머지 타입은 모두 슬롯 하나에 저장된다.
- `지역 변수 테이블`을 구성하는 데 필요한 데이터 공간은 컴파일 과정에서 할당된다. Java 메서드는 스택프레임에서 지역 변수용으로 할당받아야 할 공간의 크기(=변수 슬롯 개수)가 이미 완벽하게 결정되어 있다. 메서드 실행 중에는 절대 변하지 않는다.
- `<<자바 가상 머신 명세>>`는 스택 메모리 영역에서 두 가지 오류가 발생할 수 있도록 정의했다.
- 스레드가 요청한 스택 깊이가 가상 머신이 허용하는 깊이보다 크다면 `StackOverflowError`를 던진다.
- 스택 용량을 동적으로 확장할 수 있는 JVM에서는 스택을 확장하려는 시점에 여유 메모리가 충분하지 않다면 `OutOfMemoryError`를 던진다.

**💡네이티브 메서드 스택**

**💡자바 힙**