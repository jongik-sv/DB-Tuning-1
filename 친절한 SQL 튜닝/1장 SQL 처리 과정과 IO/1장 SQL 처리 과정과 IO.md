# 1.1 SQL 파싱과 최적화
## 1.1.1 구조적, 집합적, 선언적 질의 언어
- 구조(structured)
- 집합적(set-based)
- 선언적(declarative)

## 1.1.2 SQL 최적화
SQL 파싱과 최적화를 포함하여 `SQL 최적화`라고 표현을 많이 한다.
1) SQL 파싱
	- 파싱트리 생성
	- Syntax 체크 : 문법적 오류 확인
	- Semantic 체크 : 의미상 오류 확인, 존재하는 오브젝트/칼럼을 사용했는지, 권한이 있는지
2) SQL 최적화 : 옵티마이저가 통계정보 바탕으로 효율적인 실행 경로 선택
3) 로우 소스 생성 : 실행 가능한 코드
![[Pasted image 20250122145235.png]]

## 1.1.3 SQL 옵티마이저
1) 쿼리 수행하는데 후보군이 될만한 실행계획들을 찾아낸다.
2) 데이터 딕셔너리에 수집된 통계정보로 실행계획의 예상비용 산정
3) 최저 비용 실행계획 선택
![[Pasted image 20250122145535.png]]


## 1.1.4 실행계획과 비용
- 비용 : 쿼리 수행 시 발생할 것으로 예상되는  I/O 횟수 또는 예상 소요시간을 표현

## 1.1.5 옵티마이저 힌트
1) 힌트 사용 규칙
	- 힌트 안에 인자 나열할때 `','`(콤마) 사용 가능
	- 힌트와 힌트 사이에는 `','`(콤마) 사용 불가

```SQL
/*+ INDEX(A A_X01) INDEX(B, B_X03) */ -- 모두 유효

/*+ INDEX(C), FULL(D) */ --> 첫번째 힌트만 유효

SELECT /*+ FULL(SCOTT.EMP) */ * --무효
FROM   EMP

SELECT /*+ FULL(EMP) */ * --무효
FROM   EMP E


```

2) 힌트는 직접 힌트를 지정하면 사용한 힌트의 경로를 사용하고 나머지는 옵티마이저가 알아서 선택
3) 힌트 목록
![[Pasted image 20250122150511.png]]![[Pasted image 20250122150524.png]]


# 1.2 SQL 공유 및 재사용
## 1.2.1 소프트 파싱 VS. 하드 파싱
![[Pasted image 20250122150624.png]]

SQL 최적화 과정이 하드(HARD)한 이유
1) 예를 들어 5개의 테이블이면 조인순서의 종류만 120가지(5!)
2) 테이블당 인덱스 선택, 조인 방법까지 고려하면 어마어마하게 많음
3) 그 외 옵티마이저가 사용하는 정보가 많음
	- 테이블, 컬럼, 인덱스 정보
	- 오브젝트 통계 : 테이블 통계, 인덱스 통계, 칼럼 통계
	- 시스템 통계: CPU속도, Single Block I/O 속도, Multiblock I/O 속도
	- 옵티마이저 관련 파라메터

이렇게 어려운 작업을 거쳐 생성한 내부 프로시저를 `한번만 사용하고 버린다면 비효율`이다. 그래서  `라이브러리 캐시`가 필요하다


## 1.2.2 바인드 변수의 중요성


# 1.3 데이터 저장 구조 및 I/O 메커니즘
I/O 튜닝이 곧 SQL 튜닝

## 1.3.1 SQL이 느린 이유
![[Pasted image 20250122151358.png]]
디스크가 느리다. 메모리 보다 10000배 이상 느리다.



![[Pasted image 20250122151410.png]]
I/O가 발생하면 프로세스가 SLEEP(waiting) 상태로 I/O가 끝나서 I/O 종료 인터럽트를 받기 전까지 잔다.
일을 해야 할 프로세스가 I/O 대기로 계속 잠을 자기 때문에 성능이 느릴 수 밖에 없다.


## 1.3.2 데이터베이스 저장 구조
![[Pasted image 20250122151737.png]]

![[Pasted image 20250122151759.png]]

![[Pasted image 20250122151811.png]]

## 1.3.3 블록 단위 I/O
오라클은 기본적으로 8KB 크기의 블록을 사용(1바이트를 읽어도 8KB를 읽는다.)
![[Pasted image 20250122152049.png]]
OS도 기본 I/O 단위가 있다. (윈도우 NTFS 4KB 기본 설정)
![[Pasted image 20250122152347.png]]


## 1.3.4 시퀀셜 액세스 vs. 랜덤 액세스
![[Pasted image 20250122154532.png]]


## 1.3.5 논리적 I/O vs. 물리적 I/O

1) SGA 구조
![[Pasted image 20250122154638.png]]

2) 논리적 I/O vs. 물리적 I/O
![[Pasted image 20250122154748.png]]

물리적 I/O가 발생해도 DB 버퍼 캐시에 올려놓고 캐시에서 논리적 I/O를 같이 수행한다.
(Direct Path Read 방식으로 읽는 경우를 제외하면)

3) 버퍼캐시 히트율(Buffer Cache Hit Ratio)
		BCHR = (캐시에서 읽은 블록수 / 총 읽은 블록 수) * 100
		    = (( 논리적 I/O - 물리적 I/O) / 논리적 I/O) * 100
			= (1 - (물리적 I/O) / (논리적 I/O)) * 100
	예)
	![[Pasted image 20250122155936.png]]
		![[Pasted image 20250122160839.png]]

| 통계정보             | 설명                                        |
| ---------------- | ----------------------------------------- |
| Count            | 각 처리 단계별 실행된 수                            |
| CPU              | 각 처리 단계별 CPU 소모시간(단위:초)                   |
| Elapsed          | 각 처리 단계의 시작에서 종료까지 총 경과된 시간(단위:초)         |
| Disk(물리적 I/O)    | 각 처리 단계별 물리적인 디스크 블록 접근 횟수                |
| Query(논리적 I/O)   | 버퍼 캐시에서 데이터를 읽은 횟수입니다. 주로 최신 데이터를 읽을 때 발생 |
| Current(논리적 I/O) | 데이터 변경을 위해 필요한 읽기 횟수                      |
| Rows             | SQL 실행으로 처리된 행의 수                         |


## 1.3.6 Single Block I/O vs. Multiblock I/O
![[Pasted image 20250122161330.png]]
응답시간(response time), 처리량(throughput)

 1) Single Block I/O(소량 데이터 읽을 때 주로 사용) 
	 - 인덱스 루트 블록을 읽을 때 - ①
	 - 인덱스 루트 블록에서 얻은 주소 정보로 브랜치 블록을 읽을 때
	 - 인덱스 브랜치 블록에서 얻은 주소 정보로 리프 블록을 읽을 때 - ②
	 - 인덱스 리프 블록에서 얻은 주소 정보로 테이블 블록을 읽을 때 - ③

![[Pasted image 20250122161522.png]]

2) Multi Block I/O - 4, 5, 6, 7
	- 많은 데이터 블록을 읽을 때 효율적
	- 테이블 전체 스캔할 때 사용
	- 테이블이 클 수록 Multiblock I/O 단위가 크면 좋음(프로세스 Sleep 횟수를 줄임)

 ![[Pasted image 20250122162604.png]]

## 1.3.7 Table Full Scan vs. Index Range Scan
- 인덱스를 쓰는데도 더 느린 경우
![[Pasted image 20250122162837.png]]
많은 데이터를 읽어야 된다면 Table Full Scan을 Multiblock I/O 로 읽는다.
Index Range Scan은 랜덤 액세스로 Singleblock I/O 방식으로 디스크 블록을 읽는다.
캐시에서 블록을 못 찾으면 매번 Sleep이 필요한 I/O 매커니즘이다. 
또한 데이터 캐시는 유한 하여 LRU 방식으로 밀려나기도 한다.
그래서 **읽을 데이터가 일정량을 넘으면 인덱스 보다 Table Full Scan이 유리**하다.


## 1.3.8 캐시 탐색 메커니즘
Direct Path I/O를 제외한 모든 블록 I/O는 메모리 버퍼캐시를 경우

1) 버퍼캐시 탐색 과정
	 - 인덱스 루트 블록을 읽을 때
	 - 인덱스 루트 블록에서 얻은 주소 정보로 브랜치 블록을 읽을 때
	 - 인덱스 브랜치 블록에서 얻은 주소 정보로 리프 블록을 읽을 때
	 - 인덱스 리프 블록에서 얻은 주소 정보로 테이블 블록을 읽을 때
	- 테이블 블록을 Full Scan 할 때


2) 버퍼캐시 구조
	- 같은 입력 값은 항상 동일한 해시 체인(버킷)에 연결됨
	- 다른 입력 값이 동일한 해시 체인(버킷)에 연결될 수 있음
	- 해시 체인 내에서는 정렬이 보장되지 않음

>> 예시에서 해시 함수는 mod() 이지만 실제는훨씬 더 복잡한 알고리즘을 사용

![[Pasted image 20250122163614.png]]



3) 메모리 공유자원에 대한 액세스 직렬화 (한 놈씩 써라)
	- 버퍼캐시는 SGA 구성요소, 버퍼블록은 DBMS의 주요 공유자원
	- 두 개의 프로세스가 `동시에` 접근할 때 자원의 경합/데이터의 정합성 문제 발생 할 수 있음
	- 따라서 프로세스가 순차적으로 줄 세워서 접근해야 한다(직렬화 메커니즘이 필요하다.)
	- 참고 : 직렬화가 필요한 자바 예제
```Java
public class NonSynchronizedExample {
    private static int sharedResource = 0;

    public static void main(String[] args) {
        Thread thread1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                sharedResource++;
            }
        });

        Thread thread2 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                sharedResource--;
            }
        });

        thread1.start();
        thread2.start();

        try {
            thread1.join();
            thread2.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // 최종 값 출력, 값이 0이 안나옴, 왜 그럴까???
        System.out.println("Final value (without synchronization): " + sharedResource);
    }
}

```

4) 직렬화를 위한 Latch 와 Lock