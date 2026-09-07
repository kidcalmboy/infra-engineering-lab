# Day 2-1 — 파일 검색과 텍스트 처리 실습

> Linux 서버에서 필요한 파일과 로그 내용을 찾고, 결과를 필터링·추출·정렬·집계하는 흐름을 학습했다. 핵심은 명령어 문자열을 외우는 것이 아니라 **각 명령이 어떤 입력을 받고 어떤 출력을 만드는지** 이해하는 것이다.

## 📌 이번에 배운 내용

- `find`와 `grep`의 검색 대상 차이
- Pattern, Wildcard, Regular Expression의 기초 차이
- `grep -i`, `-n`, `-v`
- `wc -l`
- `sort -n`, `-r`
- `uniq -c`와 인접 중복
- `cut -d`, `-f`
- `awk`의 Field, Record, `$0`, `$1`, `NR`
- Pipe를 이용한 로그 분석
- 단계별 중간 결과 검증

## 📚 목차

1. 파일 검색과 내용 검색의 차이
2. Pattern과 Wildcard
3. `find`
4. `grep`
5. `wc`
6. `sort`, `uniq`, `cut`
7. `awk`
8. Pipeline 사고법
9. 실제 실습
10. 헷갈리기 쉬운 부분
11. Troubleshooting
12. 실무 포인트
13. 핵심 정리
14. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 옵션 의미 | 목적 |
|---|---|---|
| `find ~ -name "*.log"` | `-name` = 이름 일치 | 파일 이름 검색 |
| `find ~ -iname "server.log"` | `-iname` = 대소문자 무시 이름 검색 | 이름 검색 |
| `grep -i "error" file` | `-i` = ignore case | 대소문자 무시 내용 검색 |
| `grep -n "ERROR" file` | `-n` = line number | 줄 번호 표시 |
| `grep -v "INFO" file` | `-v` = invert match | 일치하지 않는 줄 출력 |
| `wc -l file` | `-l` = lines | 줄 수 계산 |
| `sort -nr` | `-n` numeric, `-r` reverse | 숫자 큰 값부터 정렬 |
| `uniq -c` | `-c` = count | 인접 중복 개수 표시 |
| `cut -d ':' -f 1` | `-d` delimiter, `-f` fields | 구분자 기준 필드 추출 |
| `awk '{print $1}'` | `$1` = 첫 필드 | 필드 기반 처리 |

---

## 1. 파일 검색과 내용 검색의 차이

### `find`

파일시스템의 **경로와 파일/디렉터리 자체**를 찾는다.

```bash
find ~ -name "*.log"
```

질문으로 표현하면:

```text
"이 이름을 가진 파일이 어디 있지?"
```

### `grep`

텍스트 입력에서 **패턴과 일치하는 줄**을 찾는다.

```bash
grep "ERROR" server.log
```

질문으로 표현하면:

```text
"이 파일 안에서 ERROR가 있는 줄은 무엇이지?"
```

따라서:

```text
find → 파일시스템 객체 검색
grep → 텍스트 내용 검색
```

---

## 2. Pattern, Wildcard, Regular Expression

초심자가 자주 혼동하는 부분이다.

### Wildcard `*`

Shell이나 일부 명령에서 `*`는 여러 문자와 매칭되는 패턴으로 사용된다.

```bash
find ~ -name "*.log"
```

여기서 `*.log`는 `.log`로 끝나는 이름을 뜻한다. 따옴표로 감싸는 이유는 Shell이 먼저 `*`를 확장하지 않고 `find`에게 패턴 자체를 전달하기 위해서다.

### Regular Expression

`grep`은 정규표현식 패턴을 사용할 수 있다.

예:

```bash
grep '^ERROR' file
```

`^`는 줄 시작을 의미하므로 `ERROR`로 시작하는 줄을 찾는다.

현재 단계에서는 복잡한 정규표현식을 외우기보다 **Wildcard와 Regex가 같은 개념이 아니라는 것**을 알아두면 된다.

---

## 3. `find`

### 기본 구조

```bash
find [검색 시작 경로] [조건]
```

예:

```bash
find ~ -name "server.log"
```

```text
~            → 검색 시작 경로
-name        → 이름 조건
"server.log" → 찾을 이름 패턴
```

### 주요 옵션/조건

- `-name`: 대소문자 구분 이름 검색
- `-iname`: 대소문자 무시 이름 검색
- `-type f`: 일반 파일만
- `-type d`: 디렉터리만

예:

```bash
find /var/log -type f -name "*.log"
```

운영에서 로그 파일 위치를 찾거나 특정 설정 파일을 추적할 때 유용하다.

---

## 4. `grep`

`grep`은 보통 **Global Regular Expression Print**로 설명한다. 입력에서 패턴에 일치하는 줄을 출력한다.

### `-i` — ignore case

```bash
grep -i "error" server.log
```

`ERROR`, `Error`, `error` 등을 대소문자 구분 없이 찾는다.

### `-n` — line number

```bash
grep -n "ERROR" server.log
```

매칭된 줄의 번호까지 표시한다. 설정 파일이나 로그에서 위치를 정확히 찾을 때 좋다.

### `-v` — invert match

```bash
grep -v "INFO" server.log
```

`INFO`와 **일치하지 않는 줄**을 출력한다.

### 자주 보게 될 추가 옵션

- `-r` / `-R`: 디렉터리 아래를 재귀적으로 검색
- `-c`: 매칭된 줄 수만 출력
- `-E`: Extended Regular Expression 사용

예:

```bash
grep -R "PermitRootLogin" /etc/ssh
```

운영에서는 설정 파일 여러 개에서 특정 설정 키를 찾을 때 자주 쓴다.

---

## 5. `wc`

`wc`는 **Word Count**다. 기본적으로 line, word, byte 등의 통계를 출력한다.

### `-l` — lines

```bash
wc -l server.log
```

줄 수를 계산한다.

Pipeline:

```bash
grep "ERROR" server.log | wc -l
```

흐름:

```text
grep → ERROR 줄만 stdout으로 출력
        ↓ pipe
wc -l → 입력된 줄 수 계산
```

즉 `wc -l`은 오류의 의미를 분석하는 것이 아니라 **들어온 레코드 수를 세는 역할**만 한다.

---

## 6. `sort`, `uniq`, `cut`

### `sort`

텍스트 줄을 정렬한다.

- `-n`: numeric sort, 숫자값으로 비교
- `-r`: reverse, 역순

```bash
sort -nr
```

숫자 기준 큰 값부터 정렬한다.

### `uniq`

`uniq`는 **인접한 동일 줄**을 처리한다.

예:

```text
web01
web02
web01
```

이 상태에서 `uniq`를 실행하면 떨어져 있는 두 `web01`은 하나로 합쳐지지 않는다.

그래서 전체 빈도 집계에서는 보통:

```bash
sort | uniq -c
```

를 사용한다.

### `-c` — count

```bash
sort servers.txt | uniq -c
```

각 연속 그룹의 개수를 앞에 표시한다.

### `cut`

구분자가 일정한 텍스트에서 필드를 빠르게 추출한다.

```bash
cut -d ':' -f 1 /etc/passwd
```

- `-d`: delimiter, 필드를 나누는 문자
- `-f`: fields, 출력할 필드 번호

예:

```text
alice:web:200
```

`:`로 나누면:

```text
1 = alice
2 = web
3 = 200
```

---

## 7. `awk`

### Record와 Field

`awk`는 입력을 **Record(보통 한 줄)**와 **Field(한 줄 안의 열)**로 나누어 처리한다.

예:

```text
10.0.0.1 GET /index.html 200 120
```

기본 공백 분리 기준에서는:

```text
$1 = 10.0.0.1
$2 = GET
$3 = /index.html
$4 = 200
$5 = 120
$0 = 전체 줄
```

`NR`은 현재까지 읽은 Record Number다.

### 기본 문법

```bash
awk '조건 {동작}' file
```

예:

```bash
awk '$4 >= 400 {print $1, $3, $4}' access.log
```

HTTP 상태 코드가 400 이상인 줄에서 IP, URL, 상태 코드를 출력한다.

### `cut`과 `awk`

```text
cut → 구조가 단순하고 필드 추출만 필요
awk → 조건 비교, 계산, 복잡한 필드 처리 필요
```

---

## 8. Pipeline 사고법

다음 명령을 통째로 외우지 않는다.

```bash
grep "ERROR" problem03.log | cut -d ':' -f 1 | sort | uniq -c | sort -nr
```

왼쪽부터 의미를 해석한다.

```text
1. ERROR 줄만 남긴다.
2. 서버 이름 필드만 뽑는다.
3. 같은 서버끼리 붙도록 정렬한다.
4. 서버별 개수를 센다.
5. 개수가 많은 순으로 정렬한다.
```

실무에서는 각 단계까지 실행하며 중간 출력이 예상과 맞는지 검증한다.

---

## 9. 🧪 실제 실습

```bash
grep "ERROR" server.log
grep "ERROR" server.log | wc -l
```

서버별 오류 집계:

```bash
grep "ERROR" problem03.log | cut -d ':' -f 1 | sort | uniq -c | sort -nr
```

느린 오류 요청:

```bash
awk '$4 >= 400 && $5 >= 100 {print $1, $3, $4, $5}' access.log
```

---

## 10. ⚠️ 헷갈리기 쉬운 부분

> ⚠️ `find`는 내용 검색 도구가 아니다. 파일 이름/경로/속성 검색에 사용한다.

> ⚠️ `uniq`는 파일 전체의 모든 중복을 자동으로 모으지 않는다. **인접 중복**만 처리한다.

> ⚠️ `sort -n`이 없으면 숫자도 문자열 순서로 정렬될 수 있다.

> ⚠️ 긴 Pipeline은 명령 하나가 아니다. 여러 Process가 Pipe로 연결된 구조다.

---

## 11. 🔧 Troubleshooting

### 증상

로그에서 어느 서버가 ERROR를 가장 많이 발생시키는지 알아야 한다.

### 확인

```bash
grep "ERROR" problem03.log
```

먼저 ERROR 줄 자체가 올바른지 확인한다.

### 분석

```text
필요한 줄 = ERROR
필요한 값 = 서버명
필요한 결과 = 서버별 빈도
```

### 조치

```bash
grep "ERROR" problem03.log | cut -d ':' -f 1 | sort | uniq -c | sort -nr
```

### 검증

Pipeline을 한 단계씩 끊어 결과를 확인한다.

---

## 12. 💼 실무 포인트

로그 분석은 다음 질문으로 접근한다.

```text
어떤 줄이 필요한가?        → grep
어떤 필드가 필요한가?      → cut / awk
같은 값을 모아야 하는가?   → sort
몇 번 발생했는가?          → uniq -c / awk
수치 우선순위가 필요한가?  → sort -nr
```

이 사고방식을 익히면 로그 형식이 달라져도 명령을 새로 조합할 수 있다.

---

## 13. ✅ 핵심 정리

- `find`는 파일시스템 객체, `grep`은 텍스트 줄을 검색한다.
- Wildcard와 Regular Expression은 같은 개념이 아니다.
- `wc -l`은 입력 줄 수를 센다.
- `uniq`는 인접 중복만 처리하므로 전체 빈도 집계에서는 `sort | uniq -c`를 자주 쓴다.
- `cut -d -f`는 구분자 기반 필드 추출에 적합하다.
- `awk`는 Record/Field 기반 조건·출력·계산에 적합하다.
- Pipeline은 작은 도구의 출력을 다음 도구의 입력으로 연결한다.

---

## 14. 🧠 복습 문제

1. `find`와 `grep`의 검색 대상 차이는?
2. `find ~ -name "*.log"`에서 따옴표를 사용하는 이유는?
3. `grep -v`는 무엇을 출력하는가?
4. `sort -nr`의 `-n`, `-r`은 각각 무엇인가?
5. 왜 `uniq -c` 전에 `sort`를 자주 사용하는가?
6. `cut -d ':' -f 2`를 해석해보라.
7. `awk`에서 `$0`, `$1`, `NR`은 무엇인가?
8. 긴 Pipeline을 단계별로 확인해야 하는 이유는?
