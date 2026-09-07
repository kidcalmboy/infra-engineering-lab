# Day 5-3 — Job Control과 `nohup`

> 현재 Shell이 관리하는 **Job**과 Kernel이 시스템 전체에서 관리하는 **Process**의 차이를 이해하고, `Ctrl+Z`, `jobs`, `bg`, `fg`로 작업 상태를 바꾸는 방법을 실습했다. 또한 Terminal/SSH Session과 SIGHUP, `nohup`, Redirection의 관계를 연결해 장기 실행 작업의 기초를 학습했다.

## 📌 이번에 배운 내용

- Process와 Job의 차이
- “Kernel이 관리”와 “Shell이 관리”의 구체적 의미
- Job ID와 PID
- Foreground / Background
- Terminal과 foreground process group 기초
- `Ctrl+C` / SIGINT
- `Ctrl+Z` / SIGTSTP
- `jobs -l`, `-r`, `-s`
- Job Specifier `%1`
- `bg`, `fg`
- `&`와 `bg`의 차이
- SIGHUP
- `nohup`
- `>`, `2>&1`, `&` 조합
- `nohup`과 systemd의 역할 차이

## 📚 목차

1. Process와 Job
2. Kernel 관리와 Shell 관리
3. Job ID와 PID
4. Foreground / Background
5. Terminal과 Signal
6. `Ctrl+C`와 `Ctrl+Z`
7. `jobs`
8. `bg`
9. `fg`
10. `&`와 `bg`
11. SIGHUP과 `nohup`
12. Redirection 조합
13. 실제 실습
14. 헷갈리기 쉬운 부분
15. Troubleshooting
16. 실무 포인트
17. 핵심 정리
18. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 옵션/표현 | 의미 |
|---|---|---|
| `jobs` | 없음 | 현재 Shell의 Job 목록 |
| `jobs -l` | `-l` = long | Job ID와 PID 함께 표시 |
| `jobs -r` | `-r` = running | Running Job만 표시 |
| `jobs -s` | `-s` = stopped | Stopped Job만 표시 |
| `bg %1` | `%1` = Job 1 | Job 1을 background에서 재개 |
| `fg %1` | `%1` = Job 1 | Job 1을 foreground로 가져옴 |
| `command &` | `&` = Shell background operator | 처음부터 background 실행 |
| `nohup command &` | no hangup | SIGHUP 영향을 피하도록 실행 |
| `nohup command > app.log 2>&1 &` | stdout/stderr + background | 출력까지 로그로 남기며 실행 |

---

## 1. Process와 Job

### Process

Program이 실행되어 Kernel이 관리하는 실행 단위다.

Kernel은 Process에 대해:

```text
PID
PPID
CPU scheduling 상태
Memory
UID/GID credential
File Descriptor
Signal 상태
```

등을 관리한다.

### Job

Job은 **현재 Shell이 사용자의 Terminal 작업을 편하게 제어하기 위해 관리하는 작업 단위**다.

예:

```bash
sleep 900 &
```

출력:

```text
[1] 2075
```

```text
[1]  → Bash가 붙인 Job ID
2075 → Kernel이 Process에 붙인 PID
```

같은 실행을 서로 다른 계층에서 보는 것이다.

---

## 2. “Kernel이 관리”와 “Shell이 관리”는 무슨 뜻인가?

### Kernel이 Process를 관리

PID 2075라는 Process는 시스템 전체에 존재한다. 같은 사용자 권한이 있고 접근 가능하다면 다른 SSH Terminal에서도:

```bash
ps -fp 2075
pgrep -a sleep
```

로 볼 수 있다.

즉 Process는 특정 Bash 창에만 존재하는 개념이 아니다.

### Shell이 Job을 관리

반면 Job ID `[1]`은 현재 Bash가 자기 내부 Job Table에 기록한 번호다.

Terminal A:

```text
Job 1 → PID 2075
```

새 Terminal B에서:

```bash
jobs
```

해도 Terminal A의 Job 1은 일반적으로 보이지 않는다.

하지만:

```bash
ps -fp 2075
```

는 볼 수 있다.

핵심:

```text
PID    → Linux 시스템 전체 범위
Job ID → 현재 Shell 범위
```

---

## 3. Job ID와 PID

Job ID는 보통:

```text
[1]
[2]
[3]
```

처럼 표시된다.

PID는:

```text
2075
2100
2101
```

같은 숫자다.

### 왜 `%1`이라고 쓰나?

Shell에서 `%`는 Job Specifier 문법에 사용된다.

```bash
fg %1
bg %2
```

즉 `%1`은 PID 1이 아니라 **Job 번호 1**이다.

반면:

```bash
kill 2075
ps -fp 2075
```

는 PID를 사용한다.

```text
fg/bg → Job ID 중심
ps/kill → PID 중심
```

---

## 4. Foreground와 Background

### Foreground

현재 Terminal의 foreground process group에 속해 사용자 입력과 Terminal-generated Signal의 주 대상이 되는 작업이다.

```bash
sleep 900
```

Shell은 foreground Job이 끝나거나 멈출 때까지 prompt를 바로 돌려주지 않는다.

### Background

Terminal을 직접 점유하지 않고 Shell이 prompt를 다시 보여준 상태에서 계속 실행되는 Job이다.

```bash
sleep 900 &
```

여기서 `&`는 Program 옵션이 아니라 **Shell operator**다.

`sleep` Program 자체가 `&`를 해석하는 것이 아니다. Bash가 background 실행 방식으로 Process를 시작한다.

---

## 5. Terminal과 foreground process group

Terminal에는 현재 foreground로 연결된 Process Group이 있다. 사용자가 `Ctrl+C`, `Ctrl+Z` 같은 키를 누르면 Terminal driver/Shell job control과 연결되어 해당 foreground group에 Signal이 전달된다.

지금 단계에서는 다음 관계를 이해하면 충분하다.

```text
Terminal
└─ Foreground Job
   └─ Ctrl+C / Ctrl+Z의 주요 대상
```

그래서 background Job은 일반적으로 같은 방식으로 `Ctrl+C`를 직접 받지 않는다.

---

## 6. `Ctrl+C`와 `Ctrl+Z`

### `Ctrl+C`

보통 foreground Job에 **SIGINT(2)**를 전달한다.

```text
INT = Interrupt
```

기본 동작을 따르는 `sleep` 같은 Process는 종료된다.

다만 SIGINT는 Process가 catch/handle할 수 있으므로 “Ctrl+C는 무조건 kill”이라고 단정하면 안 된다.

### `Ctrl+Z`

보통 foreground Job에 **SIGTSTP**를 전달해 멈추게 한다.

```text
TSTP = Terminal Stop
```

예:

```text
[1]+ Stopped sleep 900
```

Process는 사라지지 않고 **Stopped state**로 남는다.

```text
Ctrl+C → 보통 종료 요청
Ctrl+Z → 보통 일시 정지
```

---

## 7. `jobs`

### 기본

```bash
jobs
```

현재 Shell의 Job Table을 출력한다.

예:

```text
[1]+ Running sleep 900 &
```

### `jobs -l`

```bash
jobs -l
```

`-l`은 PID를 포함한 더 긴 정보를 표시한다.

실제:

```text
[1]+ 2075 Stopped sleep 900
```

이 한 줄에서:

```text
[1]    → Job ID
+      → Current/default Job 표시
2075   → PID
Stopped→ Job 상태
```

### `jobs -r`

Running Job만 표시.

### `jobs -s`

Stopped Job만 표시.

### `+`, `-`

Bash는 Job 목록에서 현재 기본 Job을 `+`, 그 다음 후보를 `-` 등으로 표시할 수 있다. 그래서 `fg`를 Job 번호 없이 실행했을 때 어떤 Job이 선택될지 이해하는 데 도움이 된다.

---

## 8. `bg`

### 한 줄 정의

Stopped Job을 **background에서 재개**한다.

```bash
bg %1
```

흐름:

```text
Job 1 = Stopped
→ bg %1
→ Running in background
```

`bg`는 이미 정상적으로 background Running 중인 Job을 “새롭게 background로 만드는” 명령이라기보다 **Stopped Job을 background에서 resume**하는 데 핵심이 있다.

---

## 9. `fg`

```bash
fg %1
```

Job 1을 foreground로 가져온다. Stopped 상태였다면 resume하면서 foreground로 올 수 있다.

그러면 Shell prompt는 해당 Job이 종료되거나 다시 멈출 때까지 돌아오지 않는다.

---

## 10. `&`와 `bg` 차이

### `&`

```bash
sleep 900 &
```

처음부터 background로 시작.

### `bg`

```text
foreground 실행
→ Ctrl+Z로 Stop
→ bg로 background resume
```

결론:

```text
&  = 시작 방식
bg = stopped Job의 재개 방식
```

---

## 11. SIGHUP과 `nohup`

### SIGHUP

SIGHUP은 역사적으로 Terminal line의 hangup을 알리는 Signal이다. Terminal/Session 종료와 연관되어 Process가 SIGHUP을 받을 수 있다.

현대 Linux의 SSH/session 관리에서는 Process lifetime이 shell option, service manager, login manager 설정 등에 따라 달라질 수 있어 “SSH 끊기면 모든 background Process가 반드시 죽는다”라고 단정하면 안 된다.

### `nohup`

`nohup`은 **no hangup**의 의미로 실행할 command가 SIGHUP을 무시하도록 준비해서 실행한다.

```bash
nohup command &
```

중요:

```text
nohup → SIGHUP 처리 관련
&     → background 실행 관련
```

둘은 서로 다른 기능이다.

`nohup`만 쓴다고 자동 background가 되는 것이 아니고, `&`만 쓴다고 SIGHUP을 무시하도록 만드는 것도 아니다.

---

## 12. `> app.log 2>&1 &` 해석

```bash
nohup command > app.log 2>&1 &
```

왼쪽부터:

```text
nohup       → command가 SIGHUP을 무시하도록 실행
command     → 실제 실행 대상
> app.log   → stdout(FD 1)을 app.log로
2>&1        → stderr(FD 2)를 현재 stdout 목적지로
&           → 전체 command를 background Job으로 시작
```

그래서:

```text
정상 출력 → app.log
오류 출력 → app.log
Terminal prompt → 즉시 반환
SIGHUP → command가 무시하도록 설정
```

### `nohup.out`

출력을 별도 redirect하지 않으면 `nohup`이 terminal output을 `nohup.out` 같은 파일로 보낼 수 있다.

실무에서는 명시적으로 log 파일을 정하는 편이 추적하기 쉽다.

---

## 13. 🧪 실제 실습

실제 실행:

```bash
sleep 900
```

`Ctrl+Z`:

```text
[1]+ Stopped sleep 900
```

확인:

```bash
jobs -l
```

실제:

```text
[1]+ 2075 Stopped sleep 900
```

해석:

```text
Job ID = 1
PID    = 2075
State  = Stopped
```

재개:

```bash
bg %1
```

확인:

```bash
jobs
```

실제:

```text
[1]+ Running sleep 900 &
```

즉 같은 PID 2075 Process가 새로 생성된 것이 아니라:

```text
Running foreground
→ Stopped
→ Running background
```

으로 상태/Job control만 바뀐 것이다.

이후:

```bash
fg %1
```

로 foreground에 다시 가져오고 `Ctrl+C`로 종료하는 흐름도 실습했다.

---

## 14. ⚠️ 헷갈리기 쉬운 부분

> ⚠️ Job과 Process는 같은 단어가 아니다. 하나의 Job이 Pipeline처럼 여러 Process로 구성될 수도 있다.

> ⚠️ `%1`은 PID가 아니라 Job Specifier다.

> ⚠️ `jobs`는 시스템 전체 Process 목록이 아니다. 현재 Shell의 Job만 본다.

> ⚠️ `Ctrl+Z`는 Process 종료가 아니라 Stop이다.

> ⚠️ `nohup`과 `&`는 같은 기능이 아니다.

> ⚠️ `nohup`은 운영 Service Manager 대체재가 아니다.

---

## 15. 🔧 Troubleshooting

### 상황

`long-task`를 foreground에서 실행했는데 종료하지 않고 Terminal prompt를 돌려받아야 한다.

### 확인/조치

```text
Ctrl+Z
→ jobs -l
→ 대상 Job ID/PID 확인
→ bg %N
→ jobs -l
```

### 검증

Job이 `Running`인지 확인하고 필요하면 Process 관점에서도:

```bash
ps -fp PID
```

로 확인한다.

### 핵심 사고방식

```text
Shell 관점이 필요한가? → jobs/bg/fg
OS Process 관점이 필요한가? → ps/pgrep/top/kill
```

---

## 16. 💼 실무 포인트

Job Control은 SSH Terminal에서 임시 작업을 다룰 때 유용하다. 하지만 장기간 운영되어야 하는 Web/DB/Agent Process는 일반적으로:

```text
systemd
Container runtime
Process supervisor
Kubernetes
```

같은 관리 계층을 사용하는 것이 적합하다.

이유:

```text
자동 시작
장애 시 재시작 정책
상태 조회
로그 관리
의존성 관리
운영 표준화
```

등이 필요하기 때문이다.

따라서:

```text
jobs/bg/fg → 현재 Shell 작업 제어
nohup      → 간단한 Session 독립 장기 작업
systemd    → 운영 Service lifecycle 관리
```

로 구분한다.

---

## 17. ✅ 핵심 정리

- Process는 Kernel의 시스템 전체 실행 단위다.
- Job은 현재 Shell의 작업 제어 단위다.
- PID는 시스템 범위, Job ID는 Shell 범위다.
- `Ctrl+C`는 보통 SIGINT, `Ctrl+Z`는 SIGTSTP와 연결된다.
- `jobs -l`은 Job ID와 PID를 함께 볼 수 있다.
- `bg`는 Stopped Job을 background에서 resume한다.
- `fg`는 Job을 foreground로 가져온다.
- `&`는 처음부터 background 실행하는 Shell operator다.
- `nohup`은 SIGHUP 처리와 관련되고 `&`는 background와 관련된다.
- 운영 Service는 `nohup`보다 systemd 같은 Service Manager가 일반적으로 적합하다.

---

## 18. 🧠 복습 문제

1. Process와 Job의 차이를 “누가 관리하는가” 관점에서 설명해보라.
2. 다른 SSH Terminal에서 PID는 보이지만 Job ID가 안 보일 수 있는 이유는?
3. `[1]+ 2075 Stopped`에서 각 값의 의미는?
4. `%1`은 무엇인가?
5. `Ctrl+C`와 `Ctrl+Z`는 각각 어떤 Signal과 연결되는가?
6. `&`와 `bg`의 차이는?
7. `nohup`과 `&`의 역할을 분리해서 설명해보라.
8. `> app.log 2>&1 &`를 왼쪽부터 해석해보라.
9. 왜 운영 Web Service를 단순 `nohup`으로만 관리하는 것이 부족한가?
