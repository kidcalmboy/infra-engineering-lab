# Day 6-5 — 직접 systemd Service 만들기와 Lifecycle 실습

> 지금까지 배운 systemd 개념을 실제로 연결하기 위해 **연습용 Script와 Service Unit을 직접 만들고, `daemon-reload → start → status → process 확인 → journal → stop → enable/disable → enable --now` 흐름을 실습**했다.

## 📌 이번에 배운 내용

- `/usr/local/bin`에 로컬 관리용 Script 배치
- Shebang(`#!/bin/bash`)
- `while true`, `echo`, `sleep`
- Script 실행 권한과 `chmod +x`
- `/etc/systemd/system/`에 직접 Service Unit 작성
- `Type=simple`
- `ExecStart=`와 실제 실행 파일 경로 관계
- `Restart=on-failure`
- `[Install]`과 `WantedBy=multi-user.target`
- `systemctl daemon-reload`
- `systemctl start`, `stop`, `status`
- `pgrep -af`를 이용한 Process 검증
- stdout → journald → journalctl 흐름
- `journalctl -u`, `-n`, `--no-pager`, `-f`
- `enable`, `disable`, `enable --now`
- `active`와 `enabled`가 서로 독립적인 상태라는 점
- enable 시 생성되는 symbolic link 관찰
- Service 이름과 Script 파일 이름이 달라도 `ExecStart=` 경로만 정확하면 된다는 점

## 📚 목차

1. 실습 목표와 구성
2. 실행 Script 만들기
3. Shebang과 무한 반복 구조
4. 실행 권한 부여
5. Service Unit 작성
6. `Type=simple`
7. `ExecStart=`
8. `Restart=on-failure`
9. `[Install]` / `WantedBy=`
10. `daemon-reload`
11. start / status / Process 검증
12. journal 로그 확인
13. stop과 종료 검증
14. enable / disable
15. symbolic link와 자동 시작
16. `enable --now`
17. 실습 중 파일명 실수에서 배운 점
18. ⚠️ 헷갈리기 쉬운 부분
19. 🔧 Troubleshooting 관점
20. 💼 실무 포인트
21. ✅ 핵심 정리
22. 🧠 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 의미 | 확인 목적 |
|---|---|---|
| `sudo nano /usr/local/bin/hello-system.sh` | 연습용 Script 작성 | systemd가 실행할 프로그램 준비 |
| `sudo chmod +x /usr/local/bin/hello-system.sh` | 실행 권한 추가 | `ExecStart=`에서 직접 실행 가능하게 함 |
| `ls -l /usr/local/bin/hello-system.sh` | 권한 확인 | `x` 비트 검증 |
| `sudo nano /etc/systemd/system/hello-systemd.service` | Service Unit 작성 | systemd 관리 규칙 정의 |
| `sudo systemctl daemon-reload` | Unit 정의 다시 읽기 | 새 Unit을 systemd에 반영 |
| `systemctl cat hello-systemd` | Unit 내용 확인 | `ExecStart`, Install 설정 검증 |
| `systemctl show hello-systemd ...` | systemd Property 확인 | Unit 인식 상태 확인 |
| `sudo systemctl start hello-systemd` | 현재 Service 시작 | runtime activation |
| `systemctl status hello-systemd` | 상태 확인 | active/Main PID/최근 로그 확인 |
| `pgrep -af hello-system` | Process 검색 | 실제 Process 존재 검증 |
| `journalctl -u hello-systemd -n 20 --no-pager` | 최근 로그 확인 | stdout이 journal에 기록되는지 확인 |
| `journalctl -u hello-systemd -f` | 실시간 로그 추적 | 10초마다 새 로그 관찰 |
| `sudo systemctl stop hello-systemd` | Service 중지 | lifecycle 종료 |
| `sudo systemctl enable hello-systemd` | 자동 시작 설정 | boot/target activation 설정 |
| `systemctl is-enabled hello-systemd` | enable 상태 확인 | enabled/disabled 확인 |
| `ls -l /etc/systemd/system/multi-user.target.wants/` | symbolic link 확인 | enable 내부 동작 관찰 |
| `sudo systemctl disable hello-systemd` | 자동 시작 해제 | symlink 연결 제거 |
| `sudo systemctl enable --now hello-systemd` | enable + start | 자동 시작과 현재 실행을 한 번에 적용 |
| `systemctl is-active hello-systemd` | 현재 활성 상태 확인 | active/inactive 확인 |

---

# 1. 실습 목표와 구성

실제 SSH 같은 중요 서비스를 수정하지 않고, 안전한 학습용 Service를 직접 만들었다.

구성:

```text
/usr/local/bin/hello-system.sh
→ 실제로 실행되는 Bash Script

/etc/systemd/system/hello-systemd.service
→ systemd가 Script를 어떻게 관리할지 정의한 Unit
```

전체 구조:

```text
systemctl
↓
systemd
↓
hello-systemd.service
↓ ExecStart
/usr/local/bin/hello-system.sh
↓
Bash Process
↓ stdout
systemd-journald
↓
journal
↓
journalctl
```

---

# 2. 실행 Script 만들기

실습에서는 다음 경로를 사용했다.

```text
/usr/local/bin/hello-system.sh
```

> 처음 계획한 이름은 `hello-systemd.sh`였지만 실제 실습에서는 `hello-system.sh`로 만들었다. **파일 이름 자체는 문제가 아니며 Unit의 `ExecStart=`와 실제 파일 경로가 정확히 일치하는지가 중요하다.**

작성 내용:

```bash
#!/bin/bash

while true
do
    echo "hello from systemd"
    sleep 10
done
```

## 왜 `/usr/local/bin`인가?

일반적인 구분:

```text
/usr/bin
→ 배포판/패키지 관리자가 제공하는 실행 파일이 주로 위치

/usr/local/bin
→ 시스템 관리자가 직접 설치하거나 만든 로컬 실행 파일을 두기 좋은 위치
```

---

# 3. Shebang과 Script 동작

## `#!/bin/bash`

### 한 줄 정의

**Shebang은 Script 파일을 직접 실행할 때 어떤 Interpreter로 해석할지 지정하는 첫 줄이다.**

```text
#!        → shebang marker
/bin/bash → 사용할 interpreter
```

## `while true`

```bash
while true
do
    ...
done
```

조건이 항상 참이므로 반복을 계속한다.

이 실습에서는 Service Process가 종료되지 않고 계속 살아 있도록 사용했다.

## `echo`

```bash
echo "hello from systemd"
```

stdout으로 메시지를 출력한다.

systemd Service의 stdout은 기본 설정에서 journal로 연결될 수 있어 `journalctl`로 확인할 수 있다.

## `sleep 10`

10초 동안 대기한다.

```text
echo
→ sleep 10
→ echo
→ sleep 10
→ 반복
```

로그가 지나치게 빠르게 쌓이지 않도록 했다.

---

# 4. 실행 권한 부여

Script가 존재하는 것과 실행 가능한 것은 별개다.

```bash
sudo chmod +x /usr/local/bin/hello-system.sh
```

`+x`는 execute 권한을 추가한다.

확인:

```bash
ls -l /usr/local/bin/hello-system.sh
```

핵심은 permission 문자열에 실행 비트 `x`가 존재하는지 확인하는 것이다.

---

# 5. Service Unit 작성

경로:

```text
/etc/systemd/system/hello-systemd.service
```

작성 내용:

```ini
[Unit]
Description=Hello Systemd Practice Service

[Service]
Type=simple
ExecStart=/usr/local/bin/hello-system.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

이 Unit은 패키지가 제공한 Service가 아니라 관리자가 직접 작성한 로컬 Unit이므로 `/etc/systemd/system/`에 배치했다.

---

# 6. `Type=simple`

### 한 줄 정의

**`Type=simple`은 `ExecStart=`로 실행한 Process를 Service의 Main Process로 간주하는 기본적인 실행 방식이다.**

이번 Script는 background로 fork하지 않고 foreground에서 계속 실행되므로 `simple`과 잘 맞는다.

```text
systemd
↓ ExecStart
hello-system.sh
↓
Bash Process 계속 실행
↓
Main Process로 추적
```

---

# 7. `ExecStart=`

```ini
ExecStart=/usr/local/bin/hello-system.sh
```

Service 시작 시 systemd가 실행할 명령 경로다.

```text
systemctl start hello-systemd
↓
systemd가 Unit 읽음
↓
ExecStart 확인
↓
/usr/local/bin/hello-system.sh 실행
```

## 파일명 실수에서 확인한 핵심

Service 이름:

```text
hello-systemd.service
```

Script 이름:

```text
hello-system.sh
```

둘은 반드시 동일할 필요가 없다.

중요한 것은:

```text
ExecStart에 적힌 경로
=
실제로 존재하고 실행 가능한 파일 경로
```

이다.

---

# 8. `Restart=on-failure`

```ini
Restart=on-failure
```

Service Process가 실패로 판단되는 방식으로 종료되면 systemd가 재시작을 시도하도록 설정한다.

반면 관리자가:

```bash
sudo systemctl stop hello-systemd
```

처럼 의도적으로 중지한 것을 단순한 프로그램 실패와 같은 것으로 보면 안 된다.

---

# 9. `[Install]`과 `WantedBy=`

```ini
[Install]
WantedBy=multi-user.target
```

`[Install]` 섹션은 주로 `enable` 시 사용할 관계를 정의한다.

```text
WantedBy=multi-user.target
↓
systemctl enable
↓
multi-user.target가 원하는 Unit으로 연결
```

---

# 10. `daemon-reload`

새 Unit 파일을 만든 뒤 실행:

```bash
sudo systemctl daemon-reload
```

의미:

```text
Disk에 새 Unit 파일 생성
↓
systemd Manager는 이전 Unit 정보만 알고 있을 수 있음
↓
daemon-reload
↓
Unit 정의 다시 읽기
```

> `daemon-reload`는 Service를 시작하지 않는다. **Unit 정의 재로딩과 Service 실행은 서로 다른 작업이다.**

---

# 11. 🧪 start / status / Process 검증

실습에서 다음 전체 흐름을 직접 수행했다.

```bash
sudo systemctl start hello-systemd
systemctl status hello-systemd
pgrep -af hello-system
```

확인 목표:

```text
start 요청 성공 여부
→ Unit Active 상태
→ Main PID
→ 실제 Process 존재 여부
```

출력 원문은 대화에 공유되지 않았으므로 특정 PID나 status 내용을 임의로 기록하지 않는다.

핵심은 **`systemctl start`가 성공했다는 사실만으로 끝내지 않고 status와 Process까지 별도로 검증했다는 점**이다.

---

# 12. journal 로그 확인

최근 로그:

```bash
journalctl -u hello-systemd -n 20 --no-pager
```

실시간 추적:

```bash
journalctl -u hello-systemd -f
```

로그 흐름:

```text
hello-system.sh
↓ echo
stdout
↓
systemd / systemd-journald
↓
journal
↓
journalctl
```

`-f`는 새 로그를 계속 follow하며 `Ctrl+C`로 종료할 수 있다.

---

# 13. stop과 종료 검증

```bash
sudo systemctl stop hello-systemd
systemctl status hello-systemd
pgrep -af hello-system
```

운영 사고방식:

```text
stop 명령 실행
≠
Process 종료가 자동으로 검증된 것
```

따라서 Service 상태와 실제 Process 존재 여부를 확인했다.

---

# 14. enable / disable

자동 시작 설정:

```bash
sudo systemctl enable hello-systemd
systemctl is-enabled hello-systemd
```

해제:

```bash
sudo systemctl disable hello-systemd
```

중요한 개념:

```text
start/stop
→ 현재 runtime 상태

enable/disable
→ 자동 activation 설정
```

따라서 다음 조합도 가능하다.

```text
inactive + enabled
active + disabled
```

이 실습을 통해 `active ≠ enabled`를 직접 lifecycle에서 확인했다.

---

# 15. Symbolic Link와 자동 시작

`enable` 후 다음 경로를 관찰했다.

```bash
ls -l /etc/systemd/system/multi-user.target.wants/
```

`WantedBy=multi-user.target`과 `enable`의 관계는 개념적으로:

```text
multi-user.target.wants/
└─ hello-systemd.service -> 실제 Unit 파일
```

처럼 symbolic link로 구성될 수 있다.

즉 enable은 "현재 Process를 실행"하는 동작이 아니라 **향후 target activation 시 Service가 함께 활성화될 수 있도록 관계를 구성하는 작업**이다.

---

# 16. `enable --now`

```bash
sudo systemctl enable --now hello-systemd
```

의미:

```text
enable
+
start
```

검증:

```bash
systemctl is-enabled hello-systemd
systemctl is-active hello-systemd
```

즉 자동 시작 설정과 현재 runtime 상태를 별도로 확인한다.

---

# 17. 실습 중 파일명 실수에서 배운 점

처음 계획:

```text
/usr/local/bin/hello-systemd.sh
```

실제 생성:

```text
/usr/local/bin/hello-system.sh
```

이 상황에서 반드시 파일 이름을 다시 바꿀 필요는 없었다.

Unit의:

```ini
ExecStart=/usr/local/bin/hello-system.sh
```

만 실제 경로와 일치하도록 만들면 된다.

### 운영 관점

이런 작은 차이는 실제 장애에서도 자주 발생할 수 있다.

```text
ExecStart 경로 오타
파일명 변경 후 Unit 미수정
실행 파일 삭제/이동
권한 누락
```

따라서 다음 확인이 중요하다.

```bash
systemctl cat UNIT
ls -l /actual/path/program
```

---

# 18. ⚠️ 헷갈리기 쉬운 부분

## Script 파일 이름과 Service 이름

> ⚠️ 둘은 같은 이름일 필요가 없다. `ExecStart=`가 올바른 실행 파일을 가리키는지가 핵심이다.

## 파일 존재와 실행 가능 여부

> ⚠️ Script가 존재해도 execute permission이 없으면 직접 실행 단계에서 문제가 날 수 있다.

## `daemon-reload`와 `start`

```text
daemon-reload → Unit 정의 다시 읽기
start         → Service 실행
```

## `start`와 `enable`

```text
start  → 지금 실행
enable → 자동 활성화 설정
```

## `Restart=on-failure`와 `systemctl stop`

> ⚠️ 자동 재시작 정책이 있다고 해서 관리자의 정상적인 stop 요청까지 무조건 다시 실행되는 것은 아니다.

## journal 로그

> ⚠️ `echo` 결과가 Terminal 화면에 직접 보여야만 로그가 남는 것은 아니다. systemd가 Service stdout을 수집해 journal에서 조회할 수 있다.

---

# 19. 🔧 Troubleshooting 관점

Service가 시작되지 않는다면 다음 흐름으로 접근한다.

```text
증상
↓
systemctl status hello-systemd
↓
journalctl -u hello-systemd
↓
systemctl cat hello-systemd
↓
ExecStart 경로 확인
↓
ls -l 실행 파일
↓
파일 존재 여부 / 실행 권한 / shebang 확인
↓
필요한 조치
↓
daemon-reload가 필요한 변경인지 확인
↓
start/restart
↓
status
↓
journal
↓
Process 및 실제 기능 검증
```

이번 파일명 차이도 바로 이 사고방식으로 해결할 수 있다.

```text
"이름이 예상과 다르다"
↓
ExecStart가 실제 경로와 일치하는가?
↓
일치하면 문제 없음
```

---

# 20. 💼 실무 포인트

직접 만든 Service를 운영할 때 다음 순서를 습관화한다.

```text
실행 프로그램 준비
→ 권한 검증
→ Unit 작성
→ Unit 내용 재확인
→ daemon-reload
→ start
→ status
→ Process
→ journal
→ 실제 기능
```

자동 시작 변경은 별도 축으로 본다.

```text
enable/disable
→ is-enabled로 검증

start/stop
→ is-active/status로 검증
```

이렇게 하면 "명령어는 성공했는데 원하는 시스템 상태가 아닌" 실수를 줄일 수 있다.

---

# 21. ✅ 핵심 정리

```text
/usr/local/bin/hello-system.sh
→ 직접 만든 실행 Script

/etc/systemd/system/hello-systemd.service
→ 직접 만든 Service Unit

Type=simple
→ ExecStart Process를 Main Process로 추적

ExecStart=
→ 실제 실행 파일 경로

Restart=on-failure
→ 실패 시 자동 재시작 정책

WantedBy=multi-user.target
→ enable 시 연결할 target 관계

daemon-reload
→ Unit 정의 다시 읽기

start / stop
→ 현재 runtime lifecycle

enable / disable
→ 자동 activation lifecycle

journalctl -u
→ Service 로그 확인
```

전체 흐름:

```text
Script 작성
→ chmod +x
→ Unit 작성
→ daemon-reload
→ start
→ status
→ pgrep
→ journalctl
→ stop
→ Process 종료 검증
→ enable/disable
→ symbolic link 관찰
→ enable --now
→ active/enabled 별도 검증
```

---

# 22. 🧠 복습 문제

1. `/usr/local/bin`과 `/usr/bin`은 운영 관점에서 어떻게 구분할 수 있는가?
2. Shebang은 무엇이며 `#!/bin/bash`는 무슨 뜻인가?
3. `while true`를 사용한 이유는 무엇인가?
4. Script가 존재하는데도 `chmod +x`가 필요한 이유는 무엇인가?
5. `hello-systemd.service`와 `hello-system.sh`의 이름이 달라도 되는 이유는 무엇인가?
6. `Type=simple`은 systemd가 Main Process를 어떻게 판단하는 방식인가?
7. `ExecStart=`가 잘못된 경로를 가리키면 어떤 종류의 장애를 예상할 수 있는가?
8. `Restart=on-failure`는 어떤 목적의 설정인가?
9. `WantedBy=multi-user.target`은 어떤 동작과 연결되는가?
10. 새 Unit 작성 후 `daemon-reload`가 필요한 이유는 무엇인가?
11. `daemon-reload`와 `start`의 차이는 무엇인가?
12. `systemctl start` 후 왜 `status`와 `pgrep`까지 확인하는가?
13. Script의 `echo` 출력이 `journalctl`에서 보이는 흐름을 설명해보자.
14. `active`와 `enabled`가 다른 개념인 이유는 무엇인가?
15. `enable --now`는 어떤 두 동작을 함께 수행하는가?
16. enable 후 `multi-user.target.wants`를 확인한 이유는 무엇인가?
17. 실무에서 Service start 실패 시 어떤 순서로 조사할 것인가?

---

## 다음 학습

다음은 이 연습용 Service를 이용해 **일부러 장애를 만들어 Troubleshooting**한다.

예정 시나리오:

```text
ExecStart 경로 오류
실행 파일 권한 오류
Service failed 상태
journal에서 원인 찾기
설정 수정
필요 시 daemon-reload
재시작
상태/로그/Process 재검증
```

목표는 단순히 정상 Service를 만드는 것이 아니라 **증상 → 가설 → 확인 → 원인 → 조치 → 검증 → 재발 방지**의 운영 절차를 실제로 반복하는 것이다.