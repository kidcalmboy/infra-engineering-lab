# Day 2 - 파일 검색과 텍스트 처리 기초

## 학습 목표

- `find`로 파일과 디렉터리를 검색한다.
- `grep`으로 파일 내용에서 필요한 문자열만 찾는다.
- `wc -l`로 결과의 줄 수를 계산한다.
- `sort`, `uniq`, `cut`으로 로그 데이터를 정렬·집계·추출한다.
- `awk`로 필드 단위 조건 처리와 필요한 값만 출력한다.
- `|` 파이프를 이용해 여러 명령어를 연결한다.
- 서버 로그에서 ERROR/WARNING 등 필요한 정보만 빠르게 추출하고 집계하는 습관을 만든다.

## 핵심 개념

### `find`와 `grep`의 차이

- `find`: **파일 자체를 찾는 명령어**
- `grep`: **파일 내용에서 문자열을 찾는 명령어**

예를 들어 `.log` 파일의 위치를 찾고 싶다면 `find`, 특정 로그 파일 안에서 `ERROR`를 찾고 싶다면 `grep`을 사용한다.

## `find` 실습

홈 디렉터리 아래에서 특정 파일을 찾았다.

```bash
find ~ -name "server.log"
```

`.log` 확장자를 가진 파일 전체 검색:

```bash
find ~ -name "*.log"
```

대소문자를 구분하지 않고 검색할 때는 `-iname`을 사용할 수 있다.

```bash
find ~ -iname "SERVER.LOG"
```

## `grep` 실습

`server.log`에서 문자열을 기준으로 필요한 줄만 추출했다.

```bash
grep "ERROR" server.log
grep "WARNING" server.log
grep -i "error" server.log
grep -n "ERROR" server.log
grep -v "INFO" server.log
```

옵션의 의미:

- `-i`: 대소문자 구분 없이 검색
- `-n`: 일치한 줄의 줄 번호도 출력
- `-v`: 해당 문자열이 **없는 줄**을 출력

실습 문제에서는 다음 명령으로 `WARNING` 로그만 확인했다.

```bash
grep "WARNING" server.log
```

`INFO`가 아닌 로그만 확인:

```bash
grep -v "INFO" server.log
```

## 파이프 `|`

파이프는 왼쪽 명령어의 출력 결과를 오른쪽 명령어의 입력으로 전달한다.

```text
명령어 1 출력 → | → 명령어 2 입력
```

예:

```bash
history | grep "cd"
```

전체 명령어 기록 중 `cd`가 포함된 줄만 출력한다.

또한 다음과 같이 로그 필터링 결과를 다른 명령으로 다시 처리할 수 있다.

```bash
grep "ERROR" server.log | wc -l
```

## `wc -l` 실습

`wc`는 입력 데이터의 개수를 계산하는 명령어이고, `-l`은 줄 수만 출력한다.

```bash
wc -l server.log
```

`grep`과 결합하면 특정 로그가 몇 줄 발생했는지 계산할 수 있다.

```bash
grep "ERROR" server.log | wc -l
```

동작 흐름:

```text
server.log
→ grep으로 ERROR 줄만 추출
→ wc -l로 추출된 줄 개수 계산
```

## 직접 실습

기존 `server.log`에 로그를 추가한 뒤 검색과 집계를 진행했다.

```bash
cd ~/linux-test

echo "ERROR disk full" >> server.log
echo "INFO cleanup started" >> server.log
echo "ERROR backup failed" >> server.log
echo "WARNING memory usage 90%" >> server.log
```

검색:

```bash
grep "ERROR" server.log
grep -n "ERROR" server.log
grep -i "error" server.log
grep -v "INFO" server.log
```

집계:

```bash
grep "ERROR" server.log | wc -l
```

명령어 기록 검색:

```bash
history | grep "cd"
```

## Troubleshooting 실습

### 상황

애플리케이션 장애가 발생했고 로그 파일의 위치가 다음과 같다고 가정했다.

```text
/var/log/app/server.log
```

요구사항은 `ERROR` 로그가 존재하는지 확인하고 총 몇 건인지 파악하는 것이다.

먼저 실제 오류 내용을 확인한다.

```bash
grep "ERROR" /var/log/app/server.log
```

그 다음 건수를 계산한다.

```bash
grep "ERROR" /var/log/app/server.log | wc -l
```

단순히 개수만 확인하는 것보다 **오류 내용 확인 → 발생 건수 계산** 순서로 접근하는 것이 원인 분석에 더 유용하다.

# `sort`, `uniq`, `cut` 로그 처리

## `sort` - 정렬

`sort`는 입력 내용을 정렬한다.

```bash
sort level.log
```

원본 파일을 직접 수정하지 않고 정렬된 결과만 출력한다.
정렬 결과를 새 파일에 저장하려면 리다이렉션을 사용한다.

```bash
sort level.log > level_sorted.log
```

숫자 기준으로 정렬할 때는 `-n`, 역순으로 정렬할 때는 `-r`을 사용한다.

```bash
sort -nr
```

로그 발생 건수처럼 숫자를 큰 순서대로 보고 싶을 때 자주 사용한다.

## `uniq` - 연속 중복 처리

`uniq`는 전체 파일에서 같은 값을 전부 찾는 것이 아니라 **서로 붙어 있는 연속된 중복값만 처리한다.**

예를 들어 입력이 다음과 같다면:

```text
1
2
1
```

`uniq` 결과도 그대로 `1, 2, 1`이다.

반대로:

```text
1
1
2
2
1
```

에서는 다음처럼 연속 중복만 제거된다.

```text
1
2
1
```

따라서 전체 데이터의 중복을 집계할 때는 보통 먼저 `sort`로 같은 값끼리 붙인다.

```bash
sort level.log | uniq
```

발생 횟수까지 확인하려면:

```bash
sort level.log | uniq -c
```

예:

```text
3 ERROR
2 INFO
1 WARNING
```

많이 발생한 순서대로 보고 싶다면:

```bash
sort level.log | uniq -c | sort -nr
```

## `cut` - 필드 추출

`cut`은 한 줄에서 필요한 필드만 추출한다.

실습 데이터:

```text
alice:web:200
bob:db:150
charlie:web:300
```

`:`를 구분자로 사용해 첫 번째 필드만 추출:

```bash
cut -d ":" -f 1 users.log
```

두 번째 필드 추출:

```bash
cut -d ":" -f 2 users.log
```

여러 필드를 동시에 추출:

```bash
cut -d ":" -f 1,2 users.log
```

- `-d`: 필드를 나누는 delimiter 지정
- `-f`: 출력할 field 번호 지정

## `cut | sort | uniq -c` 조합

`users.log`의 역할 필드가 다음처럼 나온다고 가정한다.

```text
web
db
web
```

아래 명령은 정확한 전체 집계가 되지 않을 수 있다.

```bash
cut -d ":" -f 2 users.log | uniq -c
```

`web`이 서로 떨어져 있기 때문이다.

정확히 집계하려면:

```bash
cut -d ":" -f 2 users.log | sort | uniq -c
```

발생 횟수가 많은 순으로 보려면:

```bash
cut -d ":" -f 2 users.log | sort | uniq -c | sort -nr
```

핵심 패턴:

```text
필요한 필드 추출 → 같은 값끼리 정렬 → 개수 집계 → 많은 순서로 정렬
```

## 로그 분석 실습

다음과 같은 서버 로그를 만들고 분석했다.

```text
web01:ERROR
web02:INFO
web01:ERROR
db01:WARNING
web02:ERROR
web01:ERROR
```

### 서버별 전체 로그 발생 건수

```bash
cut -d ":" -f 1 problem03.log | sort | uniq -c | sort -nr
```

### 로그 레벨별 발생 건수

```bash
cut -d ":" -f 2 problem03.log | sort | uniq -c | sort -nr
```

### ERROR가 발생한 서버별 건수

```bash
grep "ERROR" problem03.log | cut -d ":" -f 1 | sort | uniq -c | sort -nr
```

실습에서는 아래처럼 `sort`를 `cut`보다 먼저 적용하는 방식도 확인했다.

```bash
grep "ERROR" problem03.log | sort | cut -d ":" -f 1 | uniq -c | sort -nr
```

현재 데이터처럼 서버 이름이 줄의 앞부분에 있고 동일한 서버의 ERROR 로그 문자열이 같다면 이 방식도 정상적으로 집계된다. 다만 일반적으로는 **필요한 필드를 먼저 추출한 뒤 정렬하는 방식이 데이터 흐름을 이해하기 쉽다.**

## 실습에서 확인한 실수

파일 내용에서 `ERROR`를 찾으려고 아래처럼 작성하면 목적에 맞지 않는다.

```bash
find "ERROR" problem03.log
```

`find`는 파일 자체를 검색하는 명령어이고, 파일 내용 검색은 `grep`을 사용해야 한다.

```bash
grep "ERROR" problem03.log
```

또한 `uniq -c`를 사용할 때 입력값이 정렬되어 있지 않다면 같은 값이 여러 그룹으로 나뉘어 집계될 수 있다.

# `awk` 필드 기반 로그 분석

## `awk` 기본 개념

`awk`는 한 줄을 필드 단위로 나눠서 필요한 값을 출력하거나 조건에 맞는 행만 처리할 수 있다.
기본적으로 공백을 기준으로 필드를 나눈다.

예를 들어 다음 로그가 있다.

```text
10.0.0.1 GET /index.html 200 120
```

각 필드는 다음과 같다.

```text
$1 = IP
$2 = HTTP Method
$3 = URL
$4 = Status Code
$5 = Response Time
```

기본 구조:

```bash
awk '조건 {동작}' 파일
```

## 필드 출력 실습

IP 주소만 출력:

```bash
awk '{print $1}' access.log
```

IP와 URL 출력:

```bash
awk '{print $1, $3}' access.log
```

여러 필드를 필요한 순서대로 출력할 수 있다.

## 조건 필터링

GET 요청만 출력:

```bash
awk '$2 == "GET"' access.log
```

상태 코드가 200인 요청:

```bash
awk '$4 == 200' access.log
```

상태 코드가 400 이상인 요청:

```bash
awk '$4 >= 400' access.log
```

응답 시간이 100ms 이상인 요청:

```bash
awk '$5 >= 100' access.log
```

## 여러 조건 조합

AND 조건은 `&&`, OR 조건은 `||`를 사용한다.

```bash
awk '$5 >= 100 && $4 >= 400' access.log
```

위 명령은 **응답 시간이 100ms 이상이면서 상태 코드가 400 이상인 요청**만 출력한다.

404 또는 500 상태만 찾고 싶다면:

```bash
awk '$4 == 404 || $4 == 500' access.log
```

## 조건에 맞는 필드만 출력

500 에러 요청에서 IP와 URL만 출력:

```bash
awk '$4 == 500 {print $1, $3}' access.log
```

운영자가 필요한 정보만 보도록 출력 필드를 제한할 수 있다.

## Troubleshooting 실습

### 상황

웹서비스에서 에러가 발생하고 있고, 특히 응답시간이 100ms 이상인 느린 에러 요청을 확인하고 싶다고 가정했다.

조건:

```text
Status Code >= 400
Response Time >= 100
```

필요한 출력:

```text
IP URL STATUS TIME
```

실제로 작성한 명령:

```bash
awk '$4 >= 400 && $5 >= 100 {print $1, $3, $4, $5}' access.log
```

이 명령을 통해 단순히 ERROR 문자열을 찾는 것을 넘어, **로그의 필드 구조를 이해하고 숫자 조건으로 장애 요청을 추려낼 수 있었다.**

## `cut`과 `awk` 차이

- `cut`: 단순한 필드 추출에 적합
- `awk`: 필드 추출 + 조건 비교 + 계산까지 가능

예를 들어 단순히 첫 번째 필드만 가져오면 `cut`으로 충분하지만, 상태 코드와 응답시간 조건을 동시에 적용해야 한다면 `awk`가 더 적합하다.

## 배운 점

- 파일의 위치를 모르면 `find`, 파일 내용에서 값을 찾고 싶으면 `grep`을 사용한다.
- `grep` 옵션을 이용하면 대소문자 무시, 줄 번호 표시, 반대 조건 필터링이 가능하다.
- `wc -l`은 단독으로도 사용할 수 있지만 파이프와 결합할 때 특히 유용하다.
- `sort`는 데이터를 정렬하고, `uniq`는 연속된 중복값을 처리한다.
- 전체 중복 개수를 구할 때는 `sort | uniq -c` 패턴이 중요하다.
- `cut`으로 먼저 필요한 필드만 추출하면 로그 데이터를 쉽게 집계할 수 있다.
- `sort -nr`는 집계된 숫자를 큰 순서대로 정렬할 때 사용한다.
- `awk`는 `$1`, `$2`처럼 필드 번호를 사용해 원하는 값을 추출할 수 있다.
- `awk`의 `==`, `>=`, `<=`, `&&`, `||`를 이용하면 로그 조건을 직접 표현할 수 있다.
- 단순 필드 추출은 `cut`, 조건과 계산이 포함되면 `awk`가 더 적합하다.
- `|`를 이용하면 한 명령어의 결과를 다음 명령어로 넘겨 원하는 정보만 단계적으로 가공할 수 있다.
- 로그 장애 분석에서는 결과 개수만 보는 것이 아니라 실제 오류 내용과 조건을 함께 확인하는 습관이 중요하다.

앞으로 로그 분석 시 다음 흐름을 기본으로 사용한다.

```text
로그 위치 확인
→ 필요한 문자열/조건 검색
→ 필요한 필드 추출
→ 정렬 및 집계
→ 결과 해석
```
