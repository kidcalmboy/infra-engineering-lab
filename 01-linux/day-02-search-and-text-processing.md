# Day 2-1 - 파일 검색과 텍스트 처리 실습

> 파일 위치를 찾고, 로그에서 필요한 문자열과 필드를 추출한 뒤 정렬·집계하는 기본 텍스트 처리 흐름을 학습했다.

## 📌 이번에 배운 내용

- `find`와 `grep`의 역할 차이
- `grep -i`, `-n`, `-v`
- `wc -l`로 결과 개수 계산
- `sort`, `uniq`, `uniq -c`
- `cut -d`, `-f`
- `awk` 필드와 조건식
- 파이프로 명령어를 연결하는 로그 분석 흐름
- ERROR 로그를 서버별로 집계하는 Troubleshooting

## 📚 목차

1. `find`와 `grep`
2. `wc -l`
3. `sort`, `uniq`, `cut`
4. `awk`
5. 실습
6. 헷갈리기 쉬운 부분
7. Troubleshooting
8. 실무 포인트
9. 핵심 정리
10. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 용도 |
|---|---|
| `find ~ -name "*.log"` | 파일 이름 기준 검색 |
| `find ~ -iname "server.log"` | 대소문자 무시 파일 검색 |
| `grep "ERROR" server.log` | 내용에서 ERROR가 포함된 줄 검색 |
| `grep -i "error" server.log` | 대소문자 무시 검색 |
| `grep -n "ERROR" server.log` | 줄 번호와 함께 출력 |
| `grep -v "INFO" server.log` | INFO가 없는 줄 출력 |
| `wc -l file` | 줄 수 계산 |
| `sort file` | 내용 정렬 |
| `uniq -c` | 연속 중복 항목의 개수 계산 |
| `cut -d ':' -f 1 file` | `:` 기준 첫 번째 필드 추출 |
| `awk '$4 >= 400 {print $1}' file` | 조건에 맞는 필드 처리 |

---

## 1. `find`와 `grep`

### 한 줄 정의

- `find` → **파일·디렉터리 자체를 찾는 명령어**
- `grep` → **파일 내용에서 조건에 맞는 줄을 찾는 명령어**

### 예시

파일 위치 검색:

```bash
find ~ -name "server.log"
```

로그 내용 검색:

```bash
grep "ERROR" server.log
```

> ⚠️ 파일 내용에서 `ERROR`를 찾으려고 `find "ERROR" file`을 사용하는 것은 목적에 맞지 않는다.

### `grep` 옵션

| 옵션 | 의미 |
|---|---|
| `-i` | ignore case, 대소문자 무시 |
| `-n` | line number, 줄 번호 표시 |
| `-v` | invert match, 일치하지 않는 줄 출력 |

---

## 2. `wc -l`

### 한 줄 정의

`wc -l`은 입력의 **줄 수**를 계산한다.

```bash
wc -l server.log
```

`grep`과 연결하면 특정 조건의 발생 건수를 계산할 수 있다.

```bash
grep "ERROR" server.log | wc -l
```

동작:

```text
로그 전체
→ ERROR 줄만 선택
→ 선택된 줄 수 계산
```

실무에서는 개수만 먼저 보는 것보다 실제 오류 내용을 확인한 뒤 건수를 세는 것이 원인 분석에 더 도움이 된다.

---

## 3. `sort`, `uniq`, `cut`

### `sort`

입력 데이터를 정렬한다.

```bash
sort level.log
```

원본 파일은 수정하지 않고 정렬 결과를 표준 출력으로 보낸다.

숫자 역순 정렬:

```bash
sort -nr
```

- `-n` = numeric
- `-r` = reverse

### `uniq`

### 한 줄 정의

`uniq`는 **서로 붙어 있는 연속 중복**을 처리한다.

```text
1
2
1
```

은 `uniq`를 실행해도 세 줄이 그대로 남는다.

따라서 전체 중복 집계에서는 보통 먼저 `sort`한다.

```bash
sort level.log | uniq -c
```

많이 나온 순서:

```bash
sort level.log | uniq -c | sort -nr
```

### `cut`

구분자를 기준으로 필요한 필드만 추출한다.

```bash
cut -d ':' -f 1 users.log
```

| 옵션 | 의미 |
|---|---|
| `-d` | delimiter, 필드 구분자 |
| `-f` | field, 출력할 필드 번호 |

예:

```text
alice:web:200
bob:db:150
```

```bash
cut -d ':' -f 2 users.log
```

결과:

```text
web
db
```

---

## 4. `awk`

### 한 줄 정의

`awk`는 텍스트를 **필드 단위로 나누고 조건·출력·계산을 수행하는 도구**다.

공백 기준 로그:

```text
10.0.0.1 GET /index.html 200 120
```

필드:

```text
$1 = IP
$2 = Method
$3 = URL
$4 = Status Code
$5 = Response Time
```

### 기본 구조

```bash
awk '조건 {동작}' 파일
```

IP만 출력:

```bash
awk '{print $1}' access.log
```

에러 응답만:

```bash
awk '$4 >= 400' access.log
```

느린 에러 요청:

```bash
awk '$4 >= 400 && $5 >= 100 {print $1, $3, $4, $5}' access.log
```

### `cut`과 `awk` 비교

| 구분 | `cut` | `awk` |
|---|---|---|
| 단순 필드 추출 | 적합 | 가능 |
| 조건 비교 | 제한적 | 적합 |
| 숫자 계산 | X | O |
| 복잡한 로그 분석 | 제한적 | 적합 |

---

## 5. 🧪 실습

### ERROR 개수 확인

```bash
grep "ERROR" server.log
grep "ERROR" server.log | wc -l
```

### 역할별 사용자 수 집계

```bash
cut -d ':' -f 2 users.log | sort | uniq -c | sort -nr
```

### ERROR가 발생한 서버별 건수

실습 로그:

```text
web01:ERROR
web02:INFO
web01:ERROR
db01:WARNING
web02:ERROR
web01:ERROR
```

명령:

```bash
grep "ERROR" problem03.log | cut -d ':' -f 1 | sort | uniq -c | sort -nr
```

흐름:

```text
ERROR 줄 선택
→ 서버 이름 추출
→ 같은 서버끼리 정렬
→ 서버별 개수 계산
→ 많은 순으로 정렬
```

### `awk` Troubleshooting 실습

```bash
awk '$4 >= 400 && $5 >= 100 {print $1, $3, $4, $5}' access.log
```

에러이면서 느린 요청의 IP, URL, 상태 코드, 응답 시간을 추출했다.

---

## 6. ⚠️ 헷갈리기 쉬운 부분

### `uniq`는 파일 전체 중복을 자동으로 합치는가?

아니다. **연속 중복만** 처리한다.

그래서 전체 집계에서는 다음 형태를 기본으로 기억한다.

```bash
sort | uniq -c
```

### `sort`를 `cut`보다 먼저 하면 무조건 틀리는가?

아니다. 데이터 구조에 따라 정상 결과가 나올 수 있다.

다만 일반적으로 필요한 필드를 먼저 추출한 뒤 정렬하면 파이프라인의 의도가 더 명확하다.

```bash
grep "ERROR" problem03.log | cut -d ':' -f 1 | sort | uniq -c | sort -nr
```

### `find`와 `grep`의 가장 큰 차이

```text
find → 위치/이름 등 파일 자체 검색
grep → 파일 내용 검색
```

---

## 7. 🔧 Troubleshooting

### 문제

애플리케이션 장애가 발생했고 `/var/log/app/server.log`에서 ERROR가 존재하는지와 발생 건수를 확인해야 한다.

### 확인 방법

먼저 실제 메시지를 확인한다.

```bash
grep "ERROR" /var/log/app/server.log
```

그 후 개수를 계산한다.

```bash
grep "ERROR" /var/log/app/server.log | wc -l
```

### 배운 점

```text
건수만 확인
```

보다:

```text
오류 내용 확인
→ 패턴 파악
→ 건수 계산
```

순서가 운영 장애 분석에 더 적합하다.

---

## 8. 💼 실무 포인트

운영 서버의 로그 분석은 긴 명령어를 외우는 것보다 **데이터 흐름**을 이해하는 것이 중요하다.

```text
검색 → 필터링 → 필드 추출 → 정렬 → 집계 → 결과 정렬
```

명령어가 길어지면 왼쪽부터 한 단계씩 실행해서 중간 결과를 검증한다.

```bash
grep "ERROR" problem03.log
```

```bash
grep "ERROR" problem03.log | cut -d ':' -f 1
```

이런 식으로 단계별로 확인하면 잘못된 집계를 빨리 찾을 수 있다.

---

## 9. ✅ 핵심 정리

- `find`는 파일 자체 검색, `grep`은 내용 검색이다.
- `grep -v`는 일치하지 않는 줄을 출력한다.
- `wc -l`은 줄 수를 계산한다.
- `uniq`는 연속 중복만 처리하므로 전체 집계에서는 `sort | uniq -c`를 자주 사용한다.
- `cut`은 단순 필드 추출에 적합하다.
- `awk`는 필드 기반 조건 처리와 계산에 적합하다.
- 파이프라인은 왼쪽 명령의 출력을 오른쪽 명령의 입력으로 전달한다.

---

## 10. 🧠 복습 문제

1. `find`와 `grep`의 검색 대상은 어떻게 다른가?
2. `grep -v "INFO" file`은 무엇을 출력하는가?
3. `uniq`를 사용하기 전에 `sort`를 자주 실행하는 이유는 무엇인가?
4. `cut -d ':' -f 2`에서 `-d`, `-f`는 각각 무엇을 의미하는가?
5. `cut`보다 `awk`가 더 적합한 상황은 어떤 경우인가?
6. 서버별 ERROR 발생 건수를 집계하는 파이프라인의 처리 순서를 설명해보라.
