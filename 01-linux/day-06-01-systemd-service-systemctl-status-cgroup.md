# Day 6 — systemd와 서비스 관리

> Linux에서 서비스가 무엇인지부터 systemd, Unit, daemon, `systemctl`, 서비스 상태 출력 해석까지 연결해서 학습했다. 단순히 `active` 여부만 보는 것이 아니라 **Unit 정의 → 실행 상태 → 대표 프로세스 → cgroup → 최근 로그 → 실제 기능** 순서로 서비스 상태를 판단하는 관점을 익혔다.

## 📌 이번에 배운 내용

- Program / Process / Service / Daemon의 차이
- systemd가 필요한 이유와 PID 1의 의미
- systemd와 Kernel의 역할 구분
- Unit과 Unit Type, `.service`의 의미
- 일반 장기 실행 서비스와 `Type=oneshot`의 차이
- `active`와 `enabled`의 차이
- `start`, `stop`, `restart`, `reload`, `enable`, `disable`
- `static`, `masked`, `preset` 상태
- `systemctl status` 출력의 `Loaded`, `Active`, `Main PID`, `Tasks`, `Memory`, `CPU`, `CGroup`
- `ExecStartPre`, `Invocation`, `TriggeredBy`
- cgroup과 `system.slice`
- `systemctl is-active`, `systemctl is-enabled`, Exit Status
- 실제 Ubuntu Server의 `ssh.service` 상태 해석

## 📚 목차

1. Program / Process / Service / Daemon
2. systemd가 필요한 이유
3. PID 1과 systemd
4. Unit과 `.service`
5. oneshot service
6. Service 상태와 lifecycle
7. `systemctl` 기본 구조
8. 주요 `systemctl` 명령
9. `systemctl status` 출력 해석
10. cgroup과 `system.slice`
11. 실제 Ubuntu 실습 — SSH 서비스 관찰
12. 헷갈리기 쉬운 부분
13. Troubleshooting 관점
14. 실무 포인트
15. 핵심 정리
16. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 의미 | 언제 사용하는가 |
|---|---|---|
| `ps -p 1 -o pid,comm,args` | PID 1의 PID/이름/전체 명령행 확인 | 현재 init system이 무엇인지 확인 |
| `systemctl status ssh` | SSH Unit 상세 상태 조회 | 서비스 상태와 최근 로그를 함께 볼 때 |
| `systemctl is-active ssh` | 현재 활성 상태를 짧게 확인 | 사람이 빠르게 확인하거나 스크립트 조건으로 사용 |
| `systemctl is-enabled ssh` | 자동 시작 설정 확인 | 부팅 시 서비스 활성화 정책 확인 |
| `systemctl start UNIT` | 지금 Unit 시작 | 현재 런타임에서 서비스 시작 |
| `systemctl stop UNIT` | 지금 Unit 중지 | 현재 런타임에서 서비스 중지 |
| `systemctl restart UNIT` | Unit 재시작 | 프로세스를 새로 띄워야 할 때 |
| `systemctl reload UNIT` | 서비스가 지원하면 설정만 재적용 | 중단을 줄이면서 설정 반영 |
| `systemctl enable UNIT` | 자동 시작 설정 | 부팅/target 진입 시 자동 활성화 |
| `systemctl disable UNIT` | 자동 시작 해제 | 자동 활성화 연결 제거 |
| `systemctl enable --now UNIT` | enable + start | 지금 실행하고 다음 부팅에도 자동 시작 |
| `systemctl disable --now UNIT` | disable + stop | 지금 중지하고 자동 시작도 해제 |
| `echo $?` | 직전 명령의 Exit Status 확인 | 자동화/스크립트에서 성공 여부 판단 |

---

# 1. Program / Process / Service / Daemon

## Program

### 한 줄 정의

**Program은 디스크에 저장된 실행 코드다.**

예를 들어 다음은 아직 실행 중이 아니어도 파일로 존재할 수 있다.

```text
/usr/sbin/sshd
/usr/sbin/nginx
/usr/bin/python3
```

Program이 실행되면 Kernel이 그것을 Process로 관리한다.

```text
Program
↓ 실행
Process
```

## Process

### 한 줄 정의

**Process는 실행 중인 프로그램의 실행 문맥(execution context)이다.**

Kernel은 Process마다 다음과 같은 정보를 관리한다.

```text
PID / PPID
UID / GID
Virtual Memory
File Descriptor
Process State
Signal State
CPU Scheduling 정보
```

따라서 Process를 단순히 "메모리에 올라간 프로그램"이라고만 정의하면 부족하다.

## Service

### 한 줄 정의

**Service는 시스템이 지속적으로 제공해야 하는 기능을 운영 관점에서 관리하는 단위다.**

예:

```text
SSH Service   → 원격 접속 기능
Nginx Service → 웹 서버 기능
MySQL Service → 데이터베이스 기능
```

Service와 Process는 같은 개념이 아니다. 하나의 Service가 여러 Process를 포함할 수도 있다.

```text
Nginx Service
├─ master process
├─ worker process
├─ worker process
└─ worker process
```

따라서 다음처럼 구분한다.

```text
Service → 기능/운영 단위
Process → Kernel 실행 단위
```

## Daemon

### 한 줄 정의

**Daemon은 보통 백그라운드에서 장시간 실행되면서 시스템이나 사용자에게 기능을 제공하는 Process다.**

대표적인 예:

```text
sshd
cron
nginx
mysqld
```

이름 끝에 `d`가 붙는 경우가 많지만 반드시 그런 것은 아니다.

### Service와 Daemon 차이

```text
Daemon  → Process 관점
Service → 운영/관리 관점
```

예를 들어:

```text
sshd        → 실제 daemon process
ssh.service → systemd가 관리하는 service unit
```

---

# 2. systemd가 필요한 이유

서버에서 프로그램을 단순히 직접 실행하는 것만으로는 운영이 어렵다.

운영자는 보통 다음을 원한다.

```text
부팅 시 자동 실행
정상 종료
재시작
실패 여부 확인
로그 확인
의존 서비스 관리
정해진 사용자 권한으로 실행
환경 변수 적용
재시작 정책 적용
```

서비스가 많아질수록 시작 순서와 의존 관계도 중요해진다.

```text
network
↓
logging
↓
database
↓
web server
```

이러한 **서비스 생명주기와 의존성 관리**를 담당하는 것이 systemd 같은 init system / service manager다.

---

# 3. PID 1과 systemd

## systemd란?

### 한 줄 정의

**systemd는 Linux 부팅 이후 여러 시스템 리소스와 서비스를 관리하는 init system이자 service manager다.**

`init`은 initialization의 줄임말로, 시스템 초기화를 의미한다.

Ubuntu의 부팅 흐름을 단순화하면 다음과 같다.

```text
Firmware
↓
Bootloader
↓
Linux Kernel
↓
systemd (PID 1)
↓
services / user space
```

Kernel이 초기화된 뒤 최초의 user-space process를 시작하는데 Ubuntu에서는 일반적으로 systemd가 PID 1이 된다.

## PID 1이 특별한 이유

systemd는 다음과 같은 역할을 담당한다.

```text
시스템 초기화
서비스 시작/종료
서비스 상태 관리
Unit 의존 관계 관리
shutdown orchestration
고아 프로세스 수거(reaping)에 관여
```

> systemd는 Kernel이 아니다. **systemd도 user-space Process**이며 Kernel에게 Process 생성과 제어를 요청한다.

## Kernel과 systemd 역할 비교

| 구분 | Kernel | systemd |
|---|---|---|
| Process 식별 | PID/PPID 관리 | Unit과 Process의 관계 관리 |
| CPU | Scheduling | 서비스 단위 정책/상태 관리 |
| Memory | Virtual Memory 관리 | cgroup을 활용한 Unit 자원 관찰/제한 |
| Signal | Signal 전달 | 서비스 lifecycle에 맞춰 제어 요청 |
| 서비스 자동 시작 | 직접 담당하지 않음 | Unit dependency/enable 상태로 관리 |
| 재시작 정책 | 직접 서비스 정책을 갖지 않음 | `Restart=` 같은 정책 적용 가능 |

핵심은 다음과 같다.

```text
Kernel  → Process 자체를 관리
systemd → Process를 Service/Unit이라는 운영 단위로 관리
```

---

# 4. Unit과 `.service`

## Unit

### 한 줄 정의

**Unit은 systemd가 관리하는 객체의 기본 단위다.**

systemd는 서비스만 관리하지 않는다.

| Unit Type | 의미 |
|---|---|
| `.service` | 서비스 |
| `.socket` | 소켓 기반 activation |
| `.timer` | 시간 기반 실행 |
| `.mount` | 파일시스템 mount |
| `.target` | 여러 Unit을 묶는 논리적 목표 |
| `.path` | 특정 경로 변화 감시 |

현재 Day 6에서는 `.service`에 집중한다.

## `.service`

`.service` Unit은 해당 서비스를 systemd가 **어떻게 관리할지 정의**한다.

예:

```text
ssh.service
nginx.service
cron.service
```

Unit 정의에는 대략 다음과 같은 내용이 들어갈 수 있다.

```text
무엇을 실행할지
시작 전에 어떤 검사를 할지
어떤 사용자로 실행할지
무엇에 의존하는지
재시작 정책은 무엇인지
환경 변수는 무엇인지
종료 시 어떤 동작을 할지
```

Program과 Service Unit 관계는 다음처럼 볼 수 있다.

```text
ssh.service
↓
ExecStart=/usr/sbin/sshd ...
↓
systemd가 sshd Process 생성 요청
↓
Kernel이 Process 관리
```

---

# 5. oneshot service

## 한 줄 정의

**`Type=oneshot`은 한 번 정해진 작업을 실행한 뒤 Process가 종료되는 systemd 서비스 실행 방식이다.**

모든 서비스가 계속 백그라운드에 살아 있을 필요는 없다.

예:

```text
부팅 시 디렉터리 생성
초기 권한 설정
환경 초기화
특정 설정 적용
마운트 준비 작업
```

## 일반 장기 실행 서비스와 비교

일반 장기 실행 서비스:

```text
systemd
↓
nginx 시작
↓
nginx Process 계속 실행
↓
active (running)
```

oneshot 서비스:

```text
systemd
↓
초기화 작업 실행
↓
작업 완료
↓
Process 종료
```

## Unit 예시

```ini
[Service]
Type=oneshot
ExecStart=/usr/local/bin/init-task.sh
RemainAfterExit=yes
```

| 설정 | 의미 |
|---|---|
| `Type=oneshot` | 한 번 실행하고 종료되는 실행 모델 |
| `ExecStart=` | systemd가 실행할 명령 |
| `RemainAfterExit=yes` | Process 종료 후에도 Unit을 active로 간주 |

따라서 다음 상태가 가능하다.

```text
Process는 이미 종료
↓
작업은 정상 완료
↓
systemd는 Unit을 active로 기억
↓
active (exited)
```

> `Type=oneshot` 자체가 무조건 `active (exited)`를 만드는 것은 아니다. `RemainAfterExit=yes`가 있어야 종료 후에도 active 상태로 유지될 수 있다.

---

# 6. Service 상태와 Lifecycle

## `active`와 `enabled`

둘은 완전히 다른 축이다.

```text
active  → 현재 런타임 상태
enabled → 부팅/target 진입 시 자동 활성화 설정
```

그래서 네 가지 조합이 모두 가능하다.

| Active | Enabled | 의미 |
|---|---|---|
| active | enabled | 현재 실행 중 + 자동 시작 설정 |
| active | disabled | 지금은 실행 중이지만 자동 시작은 아님 |
| inactive | enabled | 지금은 꺼져 있지만 자동 시작 설정은 있음 |
| inactive | disabled | 현재도 꺼져 있고 자동 시작도 없음 |

## start / stop과 enable / disable

```text
start   → 지금 시작
stop    → 지금 중지

enable  → 자동 시작 설정
disable → 자동 시작 해제
```

따라서 `stop`했다고 다음 부팅에도 꺼져 있는 것은 아니며, `enable`했다고 지금 바로 실행되는 것도 아니다.

## restart와 reload

```text
restart → 서비스를 다시 시작
reload  → 서비스가 지원할 경우 Process를 완전히 내리지 않고 설정을 다시 읽도록 요청
```

운영 환경에서 `restart`는 연결 종료나 순간 중단을 일으킬 수 있다. 가능하다면 **서비스가 reload를 지원하는지, 재시작 영향이 무엇인지** 먼저 확인해야 한다.

---

# 7. `systemctl` 기본 구조

## 한 줄 정의

**`systemctl`은 systemd가 관리하는 Unit의 상태를 확인하고 제어하는 명령어다.**

구조:

```text
사용자
↓
systemctl
↓
systemd
↓
Unit
↓
Process
↓
Kernel
```

일반적인 문법:

```bash
systemctl ACTION UNIT
```

예:

```bash
systemctl status ssh
```

| 부분 | 의미 |
|---|---|
| `systemctl` | systemd 제어 도구 |
| `status` | 수행할 동작(Action) |
| `ssh` | 대상 Unit |

실제 Unit 이름은 `ssh.service`지만 `.service`는 생략할 수 있다.

```bash
systemctl status ssh
systemctl status ssh.service
```

두 명령은 서비스 Unit을 대상으로 할 때 사실상 같은 대상을 가리킨다.

---

# 8. 주요 `systemctl` 명령

## 상태 조회

```bash
systemctl status ssh
systemctl is-active ssh
systemctl is-enabled ssh
```

### `status`

상세 상태, 대표 Process, 자원 사용량, 최근 로그를 함께 보여준다.

### `is-active`

현재 Unit 활성 상태를 짧게 출력한다.

```text
active
inactive
failed
```

사람이 빠르게 확인하기도 좋지만 **Exit Status를 제공하므로 스크립트 조건 판단에도 유용**하다.

### `is-enabled`

자동 활성화 상태를 확인한다.

대표적인 결과:

```text
enabled
disabled
static
masked
```

## `static`

**직접 enable할 수 없는 Unit**이다. 보통 다른 Unit의 의존성 등으로 필요할 때 활성화된다.

```text
static ≠ disabled
```

## `masked`

**시작 자체를 강제로 막은 Unit 상태**다.

```text
disabled → 자동 시작만 해제, 수동 start 가능
masked   → 수동 start도 막음
```

## 실행 상태 변경

```bash
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx
```

## 자동 시작 설정

```bash
sudo systemctl enable nginx
sudo systemctl disable nginx
```

한 번에 현재 상태까지 바꾸려면:

```bash
sudo systemctl enable --now nginx
sudo systemctl disable --now nginx
```

```text
enable --now  = enable + start
disable --now = disable + stop
```

---

# 9. `systemctl status` 출력 해석

예시 형태:

```text
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/usr/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since ...
   Main PID: 1040 (sshd)
      Tasks: 1
     Memory: 7.9M
        CPU: 131ms
     CGroup: /system.slice/ssh.service
             └─1040 "sshd: /usr/sbin/sshd -D ..."
```

## `Loaded`

```text
Loaded: loaded (...)
```

systemd가 Unit 정의를 읽고 인식했다는 뜻이다.

```text
loaded ≠ running
```

경로가 함께 나오면 실제 Unit 정의 파일 위치도 알 수 있다.

```text
/usr/lib/systemd/system/ssh.service
```

## `enabled`

현재 자동 시작 설정 상태다. 현재 실행 여부와는 별개다.

## `preset: enabled`

배포판이나 패키지가 제공하는 기본 권장 정책이다.

예:

```text
disabled; preset: enabled
```

이라면 현재 관리자가 disable했지만 제공자의 기본 정책은 enabled라는 의미다.

## `Active`

```text
Active: active (running) since ...
```

- `active`: Unit이 활성 상태
- `(running)`: 일반 장기 실행 서비스에서 실제 Process가 실행 중
- `since`: 현재 상태가 언제부터 유지됐는지

장애 시점과 `since` 시간이 가까우면 서비스가 장애 직전/직후 재시작됐는지 추적하는 단서가 된다.

## `Main PID`

```text
Main PID: 1040 (sshd)
```

systemd가 해당 서비스의 대표 Process로 추적하는 PID다.

> `Main PID`는 서비스에 속한 **유일한 Process**라는 뜻이 아니다.

## `Tasks`

```text
Tasks: 1 (limit: 1688)
```

해당 Unit의 cgroup 안에 속한 task 수를 나타낸다. Linux 내부에서는 thread도 task로 다루므로 단순히 "Process 개수"라고 외우면 안 된다.

## `Memory`

```text
Memory: 7.9M (peak: 9.7M)
```

- 현재 Unit 메모리 사용량: 약 7.9 MiB
- 이번 실행에서 기록한 peak: 약 9.7 MiB

## `CPU`

```text
CPU: 131ms
```

현재 CPU 사용률이 아니라 **Unit이 사용한 누적 CPU 시간 계열 정보**다.

```text
131 ms = 0.131초
```

## `CGroup`

```text
CGroup: /system.slice/ssh.service
        └─1040 "sshd: /usr/sbin/sshd -D ..."
```

해당 Service가 속한 cgroup과 그 아래 Process를 보여준다.

---

# 10. cgroup과 `system.slice`

## cgroup

### 한 줄 정의

**cgroup(control group)은 Linux Kernel이 Process들을 그룹으로 묶어 자원 사용을 추적하거나 제한할 수 있게 하는 기능이다.**

systemd는 cgroup을 적극적으로 사용한다.

```text
system.slice
└─ ssh.service
   └─ PID 1040 sshd
```

이 방식 덕분에 서비스 단위로 다음 정보를 볼 수 있다.

```text
CPU
Memory
Tasks
서비스에 속한 Process 묶음
```

## `system.slice`

systemd의 대표적인 slice:

```text
system.slice  → 시스템 서비스
user.slice    → 사용자 세션
machine.slice → VM/container 관련
```

현재 SSH는 `system.slice` 아래의 시스템 서비스로 관리되고 있다.

---

# 11. 🧪 실제 Ubuntu 실습 — SSH 서비스 관찰

실습 환경은 SSH로 접속 중이므로 **처음부터 `stop ssh`나 `restart ssh`를 실행하지 않고 조회만 했다.** 원격 관리 서비스 자체를 실습 중단 대상으로 삼는 것은 위험할 수 있기 때문이다.

## 11-1. PID 1 확인

실행:

```bash
ps -p 1 -o pid,comm,args
```

### 옵션과 출력 항목

```text
-p 1
→ PID 1만 선택

-o
→ 출력 형식을 직접 지정

pid
→ Process ID

comm
→ 실행 파일/명령 이름

args
→ 전체 명령행
```

실제 결과:

```text
PID COMMAND         COMMAND
  1 systemd         /usr/lib/systemd/systemd --switched-root --system --deserialize=51
```

### 해석

```text
PID 1 = systemd
```

Ubuntu가 systemd를 init system/service manager로 사용하고 있음을 실제 Process 목록에서 확인했다.

뒤의 `--switched-root`, `--system`, `--deserialize=51`은 systemd 부팅 과정에서 사용되는 실행 옵션이며 현재 단계에서는 외우지 않는다.

---

## 11-2. SSH 서비스 상세 상태 확인

실행:

```bash
systemctl status ssh
```

실제 핵심 출력:

```text
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/usr/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-09-08 04:10:55 UTC; 1h 4min ago
 Invocation: 4c6521a867084e1faf441e6f9b09c294
TriggeredBy: ● ssh.socket
       Docs: man:sshd(8)
             man:sshd_config(5)
    Process: 1010 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
   Main PID: 1040 (sshd)
      Tasks: 1 (limit: 1688)
     Memory: 7.9M (peak: 9.7M)
        CPU: 131ms
     CGroup: /system.slice/ssh.service
             └─1040 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"
```

최근 로그:

```text
Sep 08 04:10:55 ubuntu-server systemd[1]: Starting ssh.service - OpenBSD Secure Shell server...
Sep 08 04:10:55 ubuntu-server sshd[1040]: Server listening on 0.0.0.0 port 22.
Sep 08 04:10:55 ubuntu-server sshd[1040]: Server listening on :: port 22.
Sep 08 04:10:55 ubuntu-server systemd[1]: Started ssh.service - OpenBSD Secure Shell server.
Sep 08 04:11:06 ubuntu-server sshd-session[1221]: Accepted password for linuxuser from 10.0.2.2 port 62753 ssh2
Sep 08 04:11:06 ubuntu-server sshd-session[1221]: pam_unix(sshd:session): session opened for user linuxuser(uid=1000)
```

### 전체 정상 흐름

```text
Unit 정의 인식
→ ExecStartPre 설정 검증 성공
→ sshd 시작
→ IPv4/IPv6 Port 22 Listen 성공
→ systemd가 서비스 시작 성공으로 판단
→ 실제 password 인증 성공
→ linuxuser 세션 생성 성공
```

단순히 `active`만 본 것이 아니라 실제 기능까지 여러 단서로 확인할 수 있었다.

---

## 11-3. `Invocation`

```text
Invocation: 4c6521a867084e1faf441e6f9b09c294
```

**Invocation ID는 systemd가 이번 Unit 실행 인스턴스를 식별하기 위해 사용하는 고유 식별자**다.

서비스를 재시작하면 새 실행 인스턴스가 되므로 Invocation ID도 달라질 수 있다.

현재 단계에서는 다음 정도로 기억한다.

```text
Invocation ID
→ "이번 ssh.service 실행"을 구분하는 ID
```

---

## 11-4. `TriggeredBy: ssh.socket`

```text
TriggeredBy: ● ssh.socket
```

`ssh.service`를 활성화할 수 있는 연결된 Socket Unit이 있다는 뜻이다.

Socket Activation 개념은 단순화하면:

```text
ssh.socket
↓
연결 요청 감지
↓
필요한 service 활성화 가능
```

`.socket`은 별도의 systemd Unit Type이며 이후 단계에서 더 깊게 다룬다.

---

## 11-5. `ExecStartPre`

```text
Process: 1010 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
```

`ExecStartPre`는 이름 그대로 **실제 서비스 시작 전 실행되는 명령**이다.

```text
Exec  → 실행
Start → 서비스 시작
Pre   → 이전
```

여기서는:

```bash
/usr/sbin/sshd -t
```

가 실행됐다.

`sshd -t`는 SSH daemon 설정을 검사한다.

```text
SSH 시작 요청
↓
sshd -t
↓
설정 검증
↓
status=0/SUCCESS
↓
실제 sshd 시작
```

`status=0/SUCCESS`는 설정 검증 명령이 성공했다는 뜻이다. 설정 파일에 심각한 문법 오류가 있다면 이 단계에서 서비스 시작이 실패할 수 있다.

---

## 11-6. IPv4와 IPv6 Listen

```text
Server listening on 0.0.0.0 port 22.
Server listening on :: port 22.
```

```text
0.0.0.0 → 모든 IPv4 로컬 인터페이스
::      → 모든 IPv6 로컬 인터페이스
```

따라서 현재 SSH daemon은 IPv4와 IPv6에서 22번 포트를 Listen하고 있다는 단서를 얻었다.

> `active` 상태는 프로세스 관점의 상태이고, 실제 Port Listen 로그는 네트워크 기능 관점의 추가 검증이다.

---

## 11-7. 실제 로그인 성공 로그

```text
Accepted password for linuxuser from 10.0.2.2 port 62753 ssh2
pam_unix(sshd:session): session opened for user linuxuser(uid=1000)
```

이 로그는 다음을 보여준다.

```text
SSH 연결 도착
→ password 인증 성공
→ linuxuser UID 1000 확인
→ 사용자 세션 생성 성공
```

VirtualBox NAT 환경에서 Host 측 연결이 Guest SSH 서비스로 정상 전달되고 있음을 실제 접속 결과로 확인했다.

---

## 11-8. `is-active`, `is-enabled`

실행:

```bash
systemctl is-active ssh
systemctl is-enabled ssh
```

결과:

```text
active
enabled
```

즉 현재 SSH는:

```text
active + enabled
```

상태다.

```text
active  → 지금 실행 중
enabled → 자동 시작 설정 존재
```

---

## 11-9. Exit Status

실행:

```bash
systemctl is-enabled ssh
echo $?
```

결과:

```text
0
```

Bash의 `$?`는 **바로 직전에 실행한 명령의 Exit Status**를 저장한다.

```text
0      → 일반적으로 성공
0 아님 → 실패 또는 조건 불충족
```

따라서 이번 `0`은 정확히 말하면 `systemctl is-enabled ssh`가 성공 상태를 반환했다는 의미다.

> `systemctl is-active ssh`의 Exit Status를 확인하려면 그 명령 바로 다음에 `echo $?`를 실행해야 한다.

---

# 12. ⚠️ 헷갈리기 쉬운 부분

## `Loaded`와 `Active`

> ⚠️ 처음에는 `Loaded: loaded`가 서비스 실행 중이라는 뜻처럼 보일 수 있다. 실제로는 **Unit 정의가 systemd에 인식됐다는 뜻**이고 현재 실행 여부는 `Active`에서 확인한다.

```text
Loaded → Unit 정의 인식
Active → 현재 Unit 상태
```

## `active`와 `enabled`

> ⚠️ `enabled`가 현재 서비스가 실행 중이라는 뜻은 아니다.

```text
active  → 현재 상태
enabled → 자동 시작 정책
```

## `Main PID`

> ⚠️ Main PID는 해당 서비스에서 유일한 Process라는 뜻이 아니다. 여러 Process를 포함하는 Service에서도 대표 Process를 하나 표시할 수 있다.

## `Tasks`

> ⚠️ `Tasks`를 단순 Process 개수와 완전히 동일하게 보면 안 된다. Linux의 task 개념에는 thread도 포함될 수 있다.

## `CPU: 131ms`

> ⚠️ 현재 CPU 사용률이 아니다. 해당 Unit이 사용한 누적 CPU 시간 계열 정보다.

## `Type=oneshot`

> ⚠️ `Type=oneshot`이라고 무조건 `active (exited)`가 되는 것은 아니다. `RemainAfterExit=yes`가 있을 때 Process 종료 후에도 active 상태를 유지할 수 있다.

## `restart`와 `reload`

> ⚠️ 둘 다 설정 반영처럼 보일 수 있지만 의미가 다르다. `restart`는 Process 중단/재생성을 일으킬 수 있고, `reload`는 서비스가 지원할 경우 Process를 유지하며 설정 재적용을 시도한다.

## `disabled`와 `masked`

```text
disabled → 자동 시작 연결 없음, 수동 start 가능
masked   → 시작 자체를 막음
```

## `$?`

> ⚠️ `$?`는 원하는 과거 명령의 결과를 기억하는 변수가 아니라 **직전 명령 하나의 Exit Status**다.

---

# 13. 🔧 Troubleshooting 관점

현재는 SSH 서비스가 정상이라 장애를 만들지 않았다. 하지만 실제 서비스 장애에서는 다음 흐름으로 접근한다.

```text
증상 확인
↓
systemctl status UNIT
↓
Loaded / Active / since / Main PID / 최근 로그 확인
↓
journalctl -u UNIT
↓
Process 확인
↓
Port/Socket 확인
↓
Config 확인
↓
원인 판단
↓
영향을 고려해 reload/restart/조치
↓
상태 재확인
↓
로그 재확인
↓
실제 기능 검증
```

## 예: `Active: failed`

단순히 "서비스가 죽었다"로 끝내지 않는다.

```text
언제 실패했는가?
↓
Result/Exit Code는 무엇인가?
↓
ExecStartPre에서 실패했는가?
↓
최근 journal에 어떤 오류가 있는가?
↓
설정 문법 문제인가?
↓
권한 문제인가?
↓
포트 충돌인가?
```

이렇게 원인을 좁혀야 한다.

---

# 14. 💼 실무 포인트

## 관리되는 서비스는 raw `kill`보다 `systemctl`을 우선 고려

특정 PID에 직접 `kill`을 보내면 Process만 종료된다. systemd가 해당 Service를 관리하고 있고 `Restart=` 정책이 있다면 다시 시작될 수도 있다.

```text
kill PID
→ Process 관점

systemctl stop UNIT
→ Service lifecycle 관점
```

운영 중인 systemd Service는 기본적으로 Service Manager를 통해 제어하는 이유다.

## SSH를 sole management path로 사용 중이면 재시작에 주의

현재 실습은 SSH로 원격 접속 중이다. 따라서 학습 초반에는 다음을 무작정 실행하지 않았다.

```bash
sudo systemctl stop ssh
sudo systemctl restart ssh
```

원격 관리 경로 자체를 중지하면 접속이 끊기거나 복구가 어려워질 수 있다.

실무에서는:

```text
대체 콘솔/관리 경로 존재 확인
→ 영향 판단
→ 설정 사전 검증
→ 변경
→ 상태/로그/접속 재검증
```

순서가 안전하다.

## `active`는 최종 성공 조건이 아니다

서비스가 `active`여도 실제 사용자가 기능을 사용하지 못할 수 있다.

예:

```text
Firewall 문제
잘못된 Listen 주소
Reverse Proxy 오류
DNS 문제
Backend 애플리케이션 오류
```

따라서 최종 검증은:

```text
Unit 상태
→ Process
→ Port
→ 로그
→ 실제 기능
```

까지 이어져야 한다.

## 시간대도 로그 분석의 일부

이번 `systemctl status`의 `since` 시각은 UTC로 표시됐다.

```text
04:10 UTC
→ 한국 시간 기준 13:10 KST
```

서비스 장애 시 사용자 신고 시간과 서버 로그 시간대가 다르면 잘못된 시점을 조사할 수 있다. 추후 `timedatectl`과 로그 시간대 관리도 연결해서 학습한다.

---

# 15. ✅ 핵심 정리

```text
Program
→ 디스크에 존재하는 실행 코드

Process
→ Kernel이 관리하는 실행 문맥

Daemon
→ 장시간 백그라운드에서 기능을 제공하는 Process

Service
→ 기능을 운영 관점에서 관리하는 단위

systemd
→ PID 1의 init system + service manager

Unit
→ systemd가 관리하는 객체

.service
→ Service 관리 정의
```

상태 개념:

```text
active  ≠ enabled
start   ≠ enable
stop    ≠ disable
restart ≠ reload
disabled ≠ masked
```

`systemctl status`에서 우선 볼 것:

```text
Unit 이름
→ Loaded
→ Active
→ since
→ Main PID
→ CGroup / Process
→ 최근 로그
```

현재 SSH 실습 결과:

```text
PID 1 = systemd
ssh.service = loaded
현재 상태 = active (running)
자동 시작 = enabled
ExecStartPre sshd -t = SUCCESS
Main PID = 1040
IPv4 0.0.0.0:22 Listen
IPv6 [::]:22 Listen
linuxuser 인증 성공
세션 생성 성공
```

즉 **서비스 정의 → 설정 검증 → Process 실행 → Port Listen → 실제 로그인 성공**까지 연결해서 확인했다.

---

# 16. 🧠 복습 문제

1. Program과 Process는 정확히 어떻게 다른가?
2. Service와 Daemon을 같은 말로 보면 안 되는 이유는 무엇인가?
3. systemd가 PID 1인 이유와 Kernel과의 역할 차이는 무엇인가?
4. Unit이란 무엇이며 `.service` 외에 어떤 Unit Type이 있는가?
5. `Type=oneshot`과 `RemainAfterExit=yes`는 각각 무엇을 의미하는가?
6. `active`와 `enabled`의 차이는 무엇인가?
7. `disabled`와 `masked`의 차이는 무엇인가?
8. `restart`와 `reload` 중 운영 중단 가능성이 더 큰 쪽은 무엇이며 왜 그런가?
9. `Loaded: loaded`만 보고 서비스를 정상 실행 중이라고 판단하면 안 되는 이유는 무엇인가?
10. `Main PID`가 해당 Service에 속한 유일한 Process라는 뜻이 아닌 이유는 무엇인가?
11. `Tasks`가 단순 Process 개수와 완전히 같지 않은 이유는 무엇인가?
12. `CPU: 131ms`는 무엇을 의미하는가?
13. cgroup은 systemd Service 관리에서 어떤 역할을 하는가?
14. `ExecStartPre=/usr/sbin/sshd -t`가 SSH 시작 전에 실행되는 이유는 무엇인가?
15. `systemctl is-enabled ssh` 후 `echo $?`가 `0`이었다면 정확히 무엇을 의미하는가?
16. 서비스가 `active`인데도 사용자가 실제 기능을 쓰지 못할 수 있는 이유를 세 가지 이상 설명해보자.

---

## 다음 학습

다음은 **`journalctl`을 이용한 systemd journal 로그 조회**로 이어진다.

예정 명령:

```bash
journalctl -u ssh -n 20 --no-pager
journalctl -u UNIT
journalctl -f
journalctl --since
journalctl -b
```

목표는 `systemctl status`에서 발견한 단서를 더 넓은 로그 범위에서 추적해 **서비스 상태 → 로그 → 원인 분석** 흐름으로 연결하는 것이다.
