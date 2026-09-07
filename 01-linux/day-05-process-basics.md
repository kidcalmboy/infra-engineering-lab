# Day 5-1~2 — Linux 프로세스 관리 (`ps`, `pgrep`, `top`, `kill`)

> Linux에서 프로그램이 실행될 때 Process가 어떻게 생성·식별·관찰·종료되는지 학습했다. 목표는 명령어 암기가 아니라 **프로세스를 찾고 → 정체를 확인하고 → 상태/자원을 보고 → 안전하게 Signal을 보내고 → 종료 여부를 검증하는 운영 절차**를 익히는 것이다.

## 📌 이번에 배운 내용

- Program / Process / Instance
- Process가 Kernel의 관리 대상이라는 의미
- PID / PPID
- Foreground / Background
- Process State 기초
- `ps`와 UNIX/BSD/GNU 옵션 스타일
- `ps -ef`, `ps aux`, `ps auf`, `ps -fp PID`
- `pgrep`, `pgrep -a`, `-f`
- `top`
- Signal
- SIGINT / SIGTERM / SIGKILL
- `kill`
- PID 재사용과 종료 전 재확인

## 📚 목차

1. Program / Process / Instance
2. Kernel이 Process를 관리한다는 의미
3. PID / PPID
4. Foreground / Background
5. Process State
6. `ps`
7. `ps -ef`
8. `ps aux` / `ps auf`
9. `ps -fp PID`
10. `pgrep`
11. `top`
12. Signal
13. `kill`, SIGTERM, SIGKILL
14. 실제 실습
15. 헷갈리기 쉬운 부분
16. Troubleshooting
17. 실무 포인트
18. 핵심 정리
19. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 옵션 의미 | 목적 |
|---|---|---|
| `sleep 500` | `500` = 대기 시간(초) | 안전한 테스트 Process 생성 |
| `sleep 500 &` | `&` = Shell background 문법 | Prompt를 즉시 돌려받음 |
| `ps` | 기본 선택 규칙 | 현재 Terminal 관련 Process 빠른 확인 |
| `ps -ef` | `-e` all, `-f` full format | PID/PPID/UID/CMD 추적 |
| `ps aux` | BSD `a/u/x` | 전체 Process + CPU/MEM/STAT 확인 |
| `ps auf` | BSD `a/u/f` | TTY Process를 사용자 형식 + forest로 확인 |
| `ps -fp 1728` | `-f` full, `-p` PID select | 특정 PID 상세 재확인 |
| `pgrep sleep` | pattern = process name | 이름으로 PID 검색 |
| `pgrep -a sleep` | `-a` = full command line | PID + 명령행 표시 |
| `pgrep -af pattern` | `-f` = full command line을 match 대상으로 사용 | 인자까지 포함해 검색 |
| `top` | 실시간 갱신 | CPU/MEM/State 관찰 |
| `kill PID` | 기본 SIGTERM | 정상 종료 요청 |
| `kill -15 PID` | 15 = SIGTERM | 정상 종료 명시 |
| `kill -9 PID` | 9 = SIGKILL | 강제 종료 |

---

## 1. Program / Process / Instance

### Program

Disk에 저장된 실행 코드와 관련 파일을 의미한다.

예:

```text
/usr/bin/sleep
```

### Process

Program이 실행되어 **Kernel이 스케줄링·메모리·Credential·열린 파일 등의 실행 상태를 관리하는 단위**다.

```bash
sleep 500
```

을 실행하면 `sleep` Program으로부터 Process가 생성된다.

### Instance

같은 Program을 여러 번 실행하면 각각 독립 Process가 된다.

```bash
sleep 700 &
sleep 900 &
```

예:

```text
sleep program
├─ PID 1718 → sleep 700
└─ PID 1728 → sleep 900
```

즉 같은 Program의 서로 다른 실행 instance다.

---

## 2. Kernel이 Process를 관리한다는 의미

“운영체제가 Process를 관리한다”는 말을 구체적으로 보면 Kernel이 각 Process에 대해 여러 실행 정보를 유지한다는 뜻이다.

예:

```text
PID / PPID
현재 상태
CPU scheduling 정보
Virtual memory/address space
UID/GID credential
열린 File Descriptor
Signal 처리 상태
```

그래서 다른 Terminal에서도 동일한 PID를 `ps`, `pgrep`, `top`으로 확인할 수 있다.

```text
Shell Job → 특정 Shell의 작업 관리 단위
Process   → Kernel이 시스템 전체에서 관리하는 실행 단위
```

Job은 Day 5-3에서 자세히 다룬다.

---

## 3. PID / PPID

### PID — Process ID

시스템에서 살아 있는 Process를 식별하는 번호다.

### PPID — Parent Process ID

해당 Process의 Parent PID다.

실습:

```text
bash PID 1348
└─ sleep PID 1401, PPID 1348
```

현재 Shell PID:

```bash
echo $$
```

`$$`는 Bash가 현재 Shell PID로 확장하는 special parameter다.

### PID는 영구 ID인가?

아니다. Process가 종료되면 PID는 이후 다른 Process에 재사용될 수 있다.

그래서 운영에서 예전에 기록한 PID만 보고 바로 `kill`하면 위험하다.

```text
PID 발견
→ 현재 command/user/parent 재확인
→ 조치
```

---

## 4. Foreground / Background

### Foreground

```bash
sleep 500
```

현재 Terminal의 foreground process group에 속해 입력/Signal을 직접 받으며 Shell은 작업이 끝날 때까지 기다린다.

### Background

```bash
sleep 500 &
```

`&`는 Bash 문법이다. 명령을 background job으로 시작하고 Shell prompt를 돌려준다.

예:

```text
[1] 1401
```

```text
[1]  → Job ID
1401 → PID
```

Job ID와 PID는 목적이 다른 번호다.

---

## 5. Process State

`ps`/`top`에서 상태 문자를 볼 수 있다.

초심자 단계 핵심:

```text
R = Running/Runnable
S = Interruptible Sleeping
D = Uninterruptible Sleep
T = Stopped/Traced
Z = Zombie
```

### `S` — Sleeping

`sleep 500`은 대부분 CPU 계산을 하는 것이 아니라 timer를 기다리므로 `S` 상태로 보일 수 있다.

### `R`

현재 CPU에서 실행 중이거나 실행 가능한 runnable 상태다.

### `Z` — Zombie

Process 실행은 끝났지만 Parent가 exit status를 아직 회수하지 않은 상태다. 메모리를 계속 크게 쓰는 “살아 있는 좀비 프로그램”으로 이해하면 안 된다.

### `D`

주로 I/O 등 Kernel 작업을 기다리는 uninterruptible sleep. `kill -9`을 보내도 즉시 사라지지 않을 수 있다. 그래서 “SIGKILL이면 무조건 즉시 없어진다”는 표현은 정확하지 않다.

---

## 6. `ps` — Process Status

`ps`는 **현재 시점의 Process table snapshot**을 출력한다.

```bash
ps
```

예:

```text
PID TTY          TIME CMD
1348 pts/0    00:00:00 bash
1401 pts/0    00:00:00 sleep
1411 pts/0    00:00:00 ps
```

주요 컬럼:

```text
PID  → Process ID
TTY  → 연결된 Terminal
TIME → 누적 CPU time
CMD  → command name
```

`TIME`은 Process가 시작된 뒤 흐른 wall-clock 시간이 아니다. CPU를 실제 사용한 누적 시간이다.

---

## 7. `ps -ef`

`ps`는 역사적으로 여러 option syntax를 지원한다.

```text
UNIX/POSIX style → -e -f
BSD style        → a u x
GNU long option  → --forest
```

```bash
ps -ef
```

- `-e`: every process, 모든 Process 선택
- `-f`: full-format listing

주요 출력:

```text
UID PID PPID C STIME TTY TIME CMD
```

특히:

```text
UID  → 실행 사용자
PID  → 현재 Process
PPID → Parent Process
CMD  → 실행 명령
```

부모-자식 관계와 실행 주체를 추적할 때 유용하다.

---

## 8. `ps aux` / `ps auf`

### `ps aux`

BSD-style options:

- `a`: 다른 사용자의 TTY 관련 Process까지 선택 범위 확대
- `u`: user-oriented output format
- `x`: controlling TTY가 없는 Process도 포함

```bash
ps aux
```

주요 컬럼:

```text
USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND
```

초심자 핵심:

- `%CPU`: CPU 사용률
- `%MEM`: 물리 Memory 비율
- `RSS`: Resident Set Size, 현재 RAM에 resident한 메모리 양에 가까운 지표
- `VSZ`: Virtual Memory Size
- `STAT`: Process state

### `ps auf`

- `a`: selection 확대
- `u`: 사용자 중심 형식
- `f`: forest ASCII hierarchy

```bash
ps auf
```

실습에서 실수로 입력했지만 유효한 명령이었다. `bash` 아래에 `sleep`이 `\_` 형태로 보여 Parent/Child 관계를 시각적으로 확인했다.

`ps auf`는 `x`가 없으므로 `ps auxf`와 선택 범위가 동일하지 않다는 점도 알아둔다.

---

## 9. `ps -fp PID`

```bash
ps -fp 1728
```

- `-f`: full-format
- `-p`: 지정 PID select

특정 Process 하나를 조치 직전에 재확인할 때 유용하다.

```text
pgrep으로 후보 찾기
→ ps -fp PID로 USER/PPID/CMD 확인
→ 조치
```

이 단계는 잘못된 Process 종료를 줄이는 데 중요하다.

---

## 10. `pgrep`

### 기본

```bash
pgrep sleep
```

Process 이름 조건으로 PID를 찾는다.

### `-a`

```bash
pgrep -a sleep
```

PID와 command line을 함께 출력한다.

실제:

```text
1718 sleep 700
1728 sleep 900
```

### `-f`

기본 `pgrep`은 Process name/comm 중심으로 match한다. 명령행 전체를 pattern 대상으로 검색하려면:

```bash
pgrep -af "sleep 900"
```

처럼 `-f`를 사용할 수 있다.

### 자주 보게 될 옵션

- `-u user`: 특정 사용자 Process
- `-P ppid`: 특정 Parent의 child Process
- `-n`: newest match
- `-o`: oldest match

운영에서는 검색 조건을 좁히는 데 유용하다.

---

## 11. `top`

`top`은 시스템 상태와 Process 목록을 **주기적으로 갱신**한다.

```bash
top
```

`ps`가 snapshot이라면 `top`은 live monitor에 가깝다.

실습에서:

```text
Tasks: ... sleeping / running / zombie
%Cpu(s): ... id
MiB Mem: ...
```

등을 확인했다.

Process row 핵심:

```text
PID USER %CPU %MEM S COMMAND
```

`q`로 종료한다.

`%CPU`는 multi-core / thread 조건에 따라 해석이 복잡해질 수 있으므로 지금 단계에서는 “어떤 Process가 상대적으로 CPU를 많이 쓰는가”를 확인하는 데 집중한다.

---

## 12. Signal

### 한 줄 정의

Signal은 Kernel/Process가 다른 Process에 특정 사건이나 제어 요청을 전달하는 비동기 메커니즘이다.

대표:

```text
SIGINT  = 2
SIGKILL = 9
SIGTERM = 15
SIGTSTP = 보통 Ctrl+Z와 연결
SIGHUP  = 1
```

Signal 번호보다 **의미와 처리 가능 여부**가 더 중요하다.

### Catch / Handle / Ignore

일부 Signal은 Process가 handler를 등록해 cleanup 후 종료하거나 무시할 수 있다.

SIGKILL과 SIGSTOP은 target Process가 catch/ignore할 수 없다.

---

## 13. `kill`, SIGTERM, SIGKILL

### `kill`

이름과 달리 핵심은 **PID에 Signal을 보내는 것**이다.

```bash
kill 1728
```

기본 Signal은 보통 SIGTERM(15).

### SIGTERM — 15

```bash
kill -15 1728
kill -TERM 1728
```

정상 종료를 요청한다. Process가 handler를 두고 있으면 자원 정리, 로그 flush, connection 종료 등을 수행할 기회를 가질 수 있다.

### SIGKILL — 9

```bash
kill -9 1728
```

Target Process가 catch/ignore할 수 없는 강제 종료 Signal이다.

하지만 D-state처럼 Kernel 내부에서 uninterruptible wait 중이면 즉시 없어지지 않을 수 있다.

### 운영 순서

```text
1. 대상 식별
2. USER/CMD/PPID/영향 확인
3. SIGTERM
4. 잠시 기다림
5. 종료 여부 검증
6. 필요하고 영향 분석됐을 때만 SIGKILL
```

---

## 14. 🧪 실제 실습

Foreground:

```bash
sleep 500
```

`Ctrl+C`로 SIGINT 전달 후 종료.

Background:

```bash
sleep 500 &
```

실제:

```text
[1] 1401
```

두 Process:

```bash
sleep 700 &
sleep 900 &
pgrep -a sleep
```

실제:

```text
1718 sleep 700
1728 sleep 900
```

상태 확인:

```bash
ps -fp 1728
top
```

종료:

```bash
kill -15 1718
pgrep -a sleep
kill -9 1728
pgrep sleep
```

마지막 `pgrep` 출력이 없어서 대상 `sleep` Process가 남지 않았음을 확인했다.

---

## 15. ⚠️ 헷갈리기 쉬운 부분

> ⚠️ Program 설치와 Process 실행은 다르다.

> ⚠️ `kill`은 본질적으로 Signal 전송 명령이다.

> ⚠️ PID는 영구적이지 않고 재사용될 수 있다.

> ⚠️ `ps TIME`은 Process가 살아온 시간과 다르다.

> ⚠️ `ps aux`의 `a/u/x`는 `aux`라는 단일 옵션이 아니다.

> ⚠️ `grep sleep` 검색에서 `grep` 자체가 보일 수 있다. `grep` 역시 실행 중인 Process이기 때문이다.

---

## 16. 🔧 Troubleshooting

### 상황

여러 `sleep` 중 `sleep 900`만 종료해야 한다.

### 증상 확인

```bash
pgrep -a sleep
```

### 대상 재확인

```bash
ps -fp 1728
```

### 상태/영향 확인

필요하면:

```bash
top
```

### 조치

```bash
kill 1728
```

### 검증

```bash
pgrep -a sleep
```

### 핵심

```text
검색 결과에서 PID를 봄
→ 바로 kill
```

이 아니라:

```text
검색
→ Identity 재확인
→ 상태 확인
→ Signal 선택
→ 종료 검증
```

순서로 접근한다.

---

## 17. 💼 실무 포인트

Process 장애 분석 질문:

```text
무슨 Process인가?
누가 실행했나?
Parent는 무엇인가?
CPU/Memory를 얼마나 쓰나?
State는 무엇인가?
어떤 Service에서 관리하나?
종료해도 되는가?
종료 후 자동 재시작되는가?
```

실제 운영 Service는 `kill`로 직접 다루기보다 systemd 같은 Service Manager를 통해 관리하는 경우가 많다. Day 6에서 이 차이를 연결한다.

---

## 18. ✅ 핵심 정리

- Process는 Kernel이 관리하는 실행 단위다.
- PID는 Process 식별자, PPID는 Parent PID다.
- `ps`는 snapshot, `top`은 live monitor다.
- `ps -ef`는 UID/PID/PPID/CMD 추적에 좋다.
- `ps aux`는 CPU/MEM/STAT 관찰에 좋다.
- `pgrep -a`는 PID와 command line을 빠르게 확인한다.
- Signal은 Process 제어/사건 전달 메커니즘이다.
- SIGTERM을 우선하고 SIGKILL은 필요한 경우에만 사용한다.
- 종료 전 대상 재확인과 종료 후 검증이 중요하다.

---

## 19. 🧠 복습 문제

1. Program과 Process 차이는?
2. Kernel이 Process를 관리한다는 말을 구체적으로 설명해보라.
3. PID와 PPID는 무엇인가?
4. `ps TIME`은 무엇을 뜻하는가?
5. `ps -ef`와 `ps aux`는 각각 무엇을 보기 좋은가?
6. `pgrep -a`와 `pgrep -af`의 차이는?
7. `S`, `R`, `D`, `T`, `Z` 상태를 설명해보라.
8. SIGTERM과 SIGKILL의 차이는?
9. 왜 `kill -9`부터 사용하면 안 되는가?
10. PID 재사용이 운영에서 왜 중요한가?
