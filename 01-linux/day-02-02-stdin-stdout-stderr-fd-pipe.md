# Day 2-2 — 표준 입출력과 텍스트 처리 파이프라인

> `grep`, `cut`, `sort`, `uniq`, `awk`를 단순 명령어 모음이 아니라 **입력 → 처리 → 출력**의 흐름으로 이해한다. 이 문서는 Linux 텍스트 처리의 기반이 되는 Standard Stream, File Descriptor, Pipe, Filter 개념을 초심자 기준으로 설명한다.

## 📌 이번에 배운 내용

- Standard Input / Output / Error
- File Descriptor 0 / 1 / 2
- Pipe가 Process 사이에서 데이터를 전달하는 방식
- Filter라는 사고방식
- Record / Field
- Filtering / Extraction / Sorting / Aggregation
- 긴 Pipeline을 단계별로 검증하는 이유

## 📚 목차

1. Standard Stream
2. File Descriptor
3. Pipe
4. Filter와 Unix 도구 철학
5. Record와 Field
6. 텍스트 처리 단계
7. 주요 명령어 역할
8. Pipeline 해석법
9. 실무 관점
10. 헷갈리기 쉬운 부분
11. Troubleshooting
12. 핵심 정리
13. 복습 문제

## ⚡ 빠른 복습

| 표현 | 의미 |
|---|---|
| `stdin` | Standard Input, 기본 입력 스트림 |
| `stdout` | Standard Output, 정상 출력 스트림 |
| `stderr` | Standard Error, 오류 출력 스트림 |
| `0` | stdin의 기본 File Descriptor |
| `1` | stdout의 기본 File Descriptor |
| `2` | stderr의 기본 File Descriptor |
| `command1 \| command2` | command1 stdout을 command2 stdin으로 연결 |
| `> file` | stdout을 파일로 보내며 덮어쓰기 |
| `>> file` | stdout을 파일 끝에 추가 |
| `2> file` | stderr만 파일로 보냄 |
| `2>&1` | stderr를 현재 stdout 목적지로 보냄 |

---

## 1. Standard Stream

Linux Process는 보통 실행될 때 세 개의 기본 입출력 통로를 가진다.

```text
stdin  → 프로그램으로 들어오는 입력
stdout → 정상 결과 출력
stderr → 오류/진단 메시지 출력
```

예를 들어:

```bash
cat server.log
```

`cat`은 파일 내용을 읽고 stdout으로 출력한다. 파일이 없으면 오류 메시지를 stderr로 출력할 수 있다.

이 구분이 중요한 이유는 정상 출력과 오류 출력을 서로 다른 곳으로 보낼 수 있기 때문이다.

---

## 2. File Descriptor

### 한 줄 정의

File Descriptor(FD)는 Process가 열린 파일·Socket·Pipe·Terminal 같은 I/O 객체를 식별하기 위해 사용하는 작은 정수다.

기본 세 개는:

```text
FD 0 = stdin
FD 1 = stdout
FD 2 = stderr
```

예:

```bash
command > result.log
```

은 사실상 stdout(FD 1)의 목적지를 Terminal 대신 `result.log`로 바꾸는 것이다.

```bash
command 2> error.log
```

은 stderr(FD 2)의 목적지만 바꾼다.

### `2>&1`

```bash
command > app.log 2>&1
```

해석:

```text
1. stdout(FD 1)을 app.log로 보냄
2. stderr(FD 2)를 현재 stdout(FD 1)이 가는 곳으로 보냄
```

따라서 정상 출력과 오류 출력이 모두 `app.log`로 들어간다.

---

## 3. Pipe `|`

### 한 줄 정의

Pipe는 **한 Process의 stdout을 다른 Process의 stdin에 연결하는 IPC(Inter-Process Communication) 방식**이다.

예:

```bash
history | grep "cd"
```

개념 구조:

```text
history Process
     stdout
       ↓
      Pipe
       ↓
     stdin
grep Process
       ↓
     stdout
```

즉 단순히 문자열을 이어 붙이는 기호가 아니라 Process 사이 데이터 전달 경로다.

### Pipe와 파일 저장 차이

```text
|  → Process → Process
>  → Process → File
```

---

## 4. Filter와 Unix 도구 철학

Linux의 많은 CLI 도구는 하나의 작업을 작게 잘 수행하고, Pipe로 서로 조합할 수 있도록 만들어져 있다.

예:

```text
grep → 줄 선택
cut  → 필드 추출
sort → 정렬
uniq → 인접 중복 처리
awk  → 조건/필드/계산
```

하나의 거대한 프로그램 대신 작은 Filter를 조합한다.

```bash
grep "ERROR" app.log | cut -d ':' -f 1 | sort | uniq -c | sort -nr
```

이 명령은 여러 Process가 Pipe로 연결된 Pipeline이다.

---

## 5. Record와 Field

텍스트 처리에서 자주 쓰는 개념이다.

### Record

보통 한 줄을 하나의 Record로 본다.

```text
10.0.0.1 GET /index.html 200 120
```

### Field

한 Record 안에서 구분자로 나뉜 값이다.

공백 기준:

```text
$1 = 10.0.0.1
$2 = GET
$3 = /index.html
$4 = 200
$5 = 120
```

이 개념은 `awk`를 이해할 때 특히 중요하다.

---

## 6. 텍스트 처리 단계

실무 로그 분석은 보통 다음 단계로 생각하면 된다.

```text
1. Filtering   → 필요한 줄만 남김
2. Extraction  → 필요한 Field만 추출
3. Sorting     → 비교/집계를 위해 정렬
4. Aggregation → 개수/합계/평균 계산
5. Ranking     → 우선순위 정렬
```

예:

```bash
grep "ERROR" problem03.log | cut -d ':' -f 1 | sort | uniq -c | sort -nr
```

```text
grep      → ERROR 줄 필터링
cut       → 서버명 추출
sort      → 같은 서버끼리 인접하게 정렬
uniq -c   → 서버별 발생 횟수 집계
sort -nr  → 숫자 기준 큰 값부터 정렬
```

---

## 7. 주요 명령어 역할

### `grep`

줄 단위 Filtering에 적합하다.

### `cut`

구조가 일정한 텍스트에서 단순 Field 추출에 적합하다.

### `sort`

줄을 정렬한다. `uniq`와 함께 쓰면 동일 값들을 인접하게 모으는 전처리 역할도 한다.

### `uniq`

서로 붙어 있는 동일 줄을 처리한다. 전체 파일에서 떨어져 있는 동일 값을 알아서 모아주는 도구는 아니다.

### `awk`

Field 기반 조건, 출력, 계산, 집계에 적합한 텍스트 처리 언어다.

---

## 8. Pipeline 해석법

긴 Pipeline을 처음부터 통째로 외우지 않는다.

다음처럼 왼쪽부터 한 단계씩 실행한다.

```bash
grep "ERROR" problem03.log
```

```bash
grep "ERROR" problem03.log | cut -d ':' -f 1
```

```bash
grep "ERROR" problem03.log | cut -d ':' -f 1 | sort
```

이렇게 하면 어느 단계에서 데이터가 예상과 달라졌는지 찾을 수 있다.

---

## 9. 💼 실무 관점

장애 분석에서는 명령어보다 **질문을 데이터 처리 단계로 번역하는 능력**이 중요하다.

예:

```text
"어느 서버에서 ERROR가 가장 많이 났나?"
```

이를 분해하면:

```text
ERROR 줄만 필요
→ 서버명만 필요
→ 서버별 묶음 필요
→ 개수 필요
→ 많은 순서 필요
```

이후 적절한 도구를 선택한다.

---

## 10. ⚠️ 헷갈리기 쉬운 부분

> ⚠️ Pipe는 파일을 저장하지 않는다. Process 간 stream을 연결한다.

> ⚠️ stdout과 stderr는 둘 다 화면에 보일 수 있지만 논리적으로 다른 stream이다.

> ⚠️ `2>&1`의 `2`와 `1`은 단순 숫자가 아니라 File Descriptor 번호다.

> ⚠️ `uniq -c`는 전체 중복 자동 집계가 아니라 인접 중복 집계다.

---

## 11. 🔧 Troubleshooting

### 증상

긴 Pipeline을 실행했는데 결과 숫자가 예상과 다르다.

### 잘못된 접근

Pipeline 전체를 반복 실행하면서 결과만 비교한다.

### 올바른 접근

```text
1단계 출력 확인
→ 2단계까지 확인
→ 3단계까지 확인
→ 처음 예상과 달라지는 지점 찾기
```

예:

```bash
grep "ERROR" file
```

```bash
grep "ERROR" file | cut -d ':' -f 1
```

중간 결과를 검증하면 잘못된 delimiter, 필드 번호, 검색 Pattern을 빠르게 발견할 수 있다.

---

## 12. ✅ 핵심 정리

- stdin=FD 0, stdout=FD 1, stderr=FD 2다.
- Pipe는 한 Process의 stdout을 다른 Process의 stdin에 연결한다.
- `>`와 Pipe는 목적이 다르다.
- Linux 텍스트 처리는 작은 Filter를 조합하는 방식이 강력하다.
- 로그 분석은 Filtering → Extraction → Sorting → Aggregation → Ranking 순서로 생각하면 쉽다.
- 긴 Pipeline은 중간 출력을 단계별로 검증한다.

---

## 13. 🧠 복습 문제

1. stdin/stdout/stderr와 FD 번호를 연결해보라.
2. `2>&1`을 말로 설명해보라.
3. Pipe와 Redirection은 어떻게 다른가?
4. Record와 Field의 차이는?
5. `sort | uniq -c`가 자주 함께 쓰이는 이유는?
6. `grep | cut | sort | uniq -c | sort -nr`의 각 단계를 설명해보라.
