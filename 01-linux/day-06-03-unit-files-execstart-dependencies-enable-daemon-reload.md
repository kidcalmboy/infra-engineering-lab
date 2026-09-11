# Day 6-3 — systemd Unit 파일 구조와 서비스 시작 과정

> `systemctl start`와 `systemctl enable`이 내부적으로 무엇을 하는지 이해하기 위해 **Unit 파일의 `[Unit]`, `[Service]`, `[Install]` 구조와 의존성, `ExecStart`, `Restart`, Target, Symbolic Link, `daemon-reload`**를 학습했다. 목표는 서비스 장애가 발생했을 때 상태와 로그뿐 아니라 **실제 Unit 정의까지 추적할 수 있는 운영 관점**을 만드는 것이다.

## 📌 이번에 배운 내용

- Unit 파일의 역할과 `.service` 파일의 의미
- `[Unit]`, `[Service]`, `[Install]` 섹션 역할
- `Description=`, `After=`, `Before=`
- `Requires=`와 `Wants=`의 차이
- **의존 관계와 시작 순서는 서로 다른 개념**이라는 점
- `Type=`과 서비스 시작 완료 판단 방식
- `ExecStartPre=`, `ExecStart=`, `ExecStop=`, `ExecReload=`
- `Restart=` 정책
- `.target` Unit과 `multi-user.target`
- `systemctl enable`과 Symbolic Link의 관계
- `/usr/lib/systemd/system/`, `/etc/systemd/system/`, `/run/systemd/system/`
- `systemctl daemon-reload`
- `systemctl cat`, `list-units`, `list-unit-files`

## 📚 목차

1. Unit 파일이란?
2. Unit 파일의 세 가지 주요 섹션
3. `[Unit]` — 설명, 의존 관계, 시작 순서
4. `[Service]` — Process 실행과 lifecycle
5. `Type=` — 서비스 시작 완료 판단 방식
6. `ExecStartPre` / `ExecStart` / `ExecStop` / `ExecReload`
7. `Restart=` 정책
8. `[Install]`과 Target
9. `enable`과 Symbolic Link
10. Unit 파일 위치와 Override
11. `daemon-reload`
12. 주요 조회 명령
13. 헷갈리기 쉬운 부분
14. 실무 운영 흐름
15. 핵심 정리
16. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 의미 | 언제 사용하는가 |
|---|---|---|
| `systemctl cat ssh` | Unit 원본과 적용된 drop-in/override 확인 | 실제 적용되는 Unit 정의 확인 |
| `systemctl show ssh -p FragmentPath` | Unit 원본 파일 경로 확인 | Unit 파일 위치 추적 |
| `systemctl show ssh -p UnitFileState` | enable 관련 상태 확인 | enabled/disabled/static 등 확인 |
| `systemctl show ssh -p ActiveState -p SubState` | Unit의 논리 상태와 세부 상태 확인 | 상태를 속성 단위로 확인 |
| `systemctl list-units` | 현재 systemd가 로드한 Unit 조회 | 현재 런타임 Unit 관찰 |
| `systemctl list-unit-files` | 설치된 Unit 파일과 enable 상태 조회 | 서비스 설치/자동 시작 상태 점검 |
| `sudo systemctl daemon-reload` | systemd Manager가 Unit 정의를 다시 읽음 | Unit 파일 또는 drop-in 변경 후 |

---

# 1. Unit 파일이란?

## 한 줄 정의

**Unit 파일은 systemd에게 특정 Unit을 어떻게 관리해야 하는지 알려주는 설정 파일이다.**

예를 들어 `ssh.service`는 SSH 실행 파일 자체가 아니다. `sshd`를 **어떤 조건과 순서로 시작하고, 어떤 명령으로 실행하고, 어떻게 종료·재시작하며, enable 시 어떤 Target과 연결할지** 정의하는 운영 규칙이다.

```text
Program
/usr/sbin/sshd

        ↑ 실제 실행 대상

Unit File
ssh.service

        ↑ 실행/의존성/lifecycle 규칙

systemd
```

---

# 2. Unit 파일의 세 가지 주요 섹션

Service Unit에서는 주로 다음 세 섹션을 만난다.

```ini
[Unit]

[Service]

[Install]
```

| 섹션 | 핵심 질문 |
|---|---|
| `[Unit]` | 이 Unit은 무엇이며 다른 Unit과 어떤 관계인가? |
| `[Service]` | 실제 Process를 어떻게 실행하고 관리할 것인가? |
| `[Install]` | `enable` 시 어떤 Target/Unit과 연결할 것인가? |

이 세 영역을 분리해서 읽으면 처음 보는 Unit 파일도 구조적으로 해석하기 쉬워진다.

---

# 3. `[Unit]` — 설명, 의존 관계, 시작 순서

예:

```ini
[Unit]
Description=Example Service
Requires=database.service
After=network.target database.service
```

## `Description=`

사람이 읽기 위한 Unit 설명이다. `systemctl status`의 첫 줄에서 볼 수 있는 설명과 연결된다.

## `After=` / `Before=`

**Ordering Dependency**, 즉 시작·종료 순서를 정의한다.

```ini
After=database.service
```

은 시작 순서상 현재 Unit이 `database.service` 뒤에 배치된다는 뜻이다.

> ⚠️ `After=`는 그 Unit을 자동으로 시작시키는 의존성을 만드는 설정이 아니다.

```text
After=
→ 순서 관계

Requires= / Wants=
→ 활성화 의존 관계
```

## `Requires=`

상대적으로 강한 의존 관계다. 현재 Unit이 활성화될 때 지정한 Unit도 함께 활성화하도록 요구한다. 필요한 Unit의 활성화 실패가 현재 Unit의 시작에도 영향을 줄 수 있다.

## `Wants=`

`Requires=`보다 약한 의존 관계다. 지정한 Unit을 함께 활성화하려고 하지만, 그 Unit의 실패를 현재 Unit의 실패와 강하게 묶지 않는다.

### 비교

| 설정 | 의미 |
|---|---|
| `Requires=` | 강한 활성화 의존 관계 |
| `Wants=` | 약한 활성화 의존 관계 |
| `After=` | 시작 순서상 뒤 |
| `Before=` | 시작 순서상 앞 |

실제 Unit에서는 **의존성과 Ordering을 함께 작성하는 경우가 많다.**

```ini
Requires=database.service
After=database.service
```

이 경우:

```text
database.service가 필요하고
+
시작 순서도 database 뒤
```

라는 두 가지 의미를 각각 표현한다.

---

# 4. `[Service]` — Process 실행과 Lifecycle

예:

```ini
[Service]
Type=notify
ExecStartPre=/usr/sbin/sshd -t
ExecStart=/usr/sbin/sshd -D
Restart=on-failure
```

`[Service]`는 실제 Process 실행과 가장 직접적으로 연결된다.

```text
systemctl start UNIT
↓
systemd가 Unit 정의 확인
↓
[Service] 설정에 따라 Process 실행
↓
Kernel이 Process 생성/스케줄링/메모리 관리
↓
systemd가 Unit과 cgroup 관점에서 추적
```

---

# 5. `Type=` — 서비스 시작 완료 판단 방식

## 한 줄 정의

**`Type=`은 systemd가 해당 Service의 시작 과정을 어떻게 해석하고, 언제 시작이 완료됐다고 판단할지 결정하는 설정이다.**

대표적인 Type:

| Type | 핵심 개념 |
|---|---|
| `simple` | 시작한 Process를 바로 Main Process로 간주 |
| `exec` | 실행 파일 실행 성공 단계까지 확인 |
| `oneshot` | 한 번 작업을 수행하고 종료 |
| `forking` | 전통적인 daemon의 fork/background 방식 |
| `notify` | Service가 systemd에 준비 완료를 알림 |

`oneshot`은 이전 파트에서 학습했다. `RemainAfterExit=yes`와 함께 사용하면 작업 Process 종료 뒤에도 `active (exited)` 상태를 유지할 수 있다.

> 모든 Type의 세부 구현을 외우는 것보다 **systemd가 서비스 시작 완료를 판단하는 방식이 여러 가지라는 점**을 이해하는 것이 먼저다.

---

# 6. 실행 관련 Directive

## `ExecStartPre=`

실제 `ExecStart=` 이전에 실행할 명령이다.

SSH 실습에서 확인한 예:

```ini
ExecStartPre=/usr/sbin/sshd -t
```

`sshd -t`로 설정을 먼저 검증한다.

```text
start 요청
↓
ExecStartPre
↓
설정 검증
├─ 실패 → 서비스 시작 중단 가능
└─ 성공
   ↓
ExecStart
```

## `ExecStart=`

Service의 핵심 실행 명령이다.

```ini
ExecStart=/usr/sbin/sshd -D
```

따라서 다음 흐름으로 연결된다.

```text
systemctl start ssh
↓
systemd가 ssh.service 확인
↓
ExecStart 확인
↓
/usr/sbin/sshd -D 실행
↓
Kernel이 sshd Process 관리
```

## `ExecStop=`

정의되어 있다면 서비스 종료 시 사용할 명령을 지정할 수 있다. 모든 Unit이 반드시 `ExecStop=`을 직접 정의하는 것은 아니다.

## `ExecReload=`

서비스 설정 재적용 방법을 정의할 수 있다.

```text
systemctl reload UNIT
↓
Unit이 reload 방법을 지원하는가?
↓
ExecReload 또는 해당 서비스의 reload 동작 수행
```

따라서 **모든 Service가 `reload`를 지원하는 것은 아니다.**

---

# 7. `Restart=` 정책

`Restart=`는 Service Process가 종료됐을 때 systemd가 자동 재시작할지를 결정하는 정책이다.

대표적인 값:

```text
Restart=no
→ 자동 재시작하지 않음

Restart=on-failure
→ 실패로 판단되는 종료에서 재시작

Restart=always
→ 정책상 재시작 대상이 되는 종료에서 지속적으로 다시 시작
```

이 개념은 Process와 Service 관리 차이를 보여준다.

```text
kill PID
↓
현재 Process 종료
↓
systemd가 종료 감지
↓
Restart 정책 확인
↓
새 Process를 다시 생성할 수도 있음
```

따라서 systemd가 관리하는 서비스는 보통 raw `kill`보다 `systemctl`을 이용해 lifecycle 관점에서 제어하는 것이 안전하다.

---

# 8. `[Install]`과 Target

예:

```ini
[Install]
WantedBy=multi-user.target
```

## `.target`이란?

**여러 Unit을 논리적으로 묶거나 특정 시스템 상태를 표현하는 systemd Unit Type**이다.

`multi-user.target`은 일반적인 서버의 multi-user 운영 상태를 표현하는 대표 Target이다.

`WantedBy=`는 Unit을 `enable`할 때 어떤 Target과 연결할지 정의하는 데 사용된다.

---

# 9. `enable`과 Symbolic Link

`systemctl enable`은 단순히 "enabled=true" 같은 값을 저장하는 명령으로만 이해하면 부족하다.

systemd에서는 일반적으로 Target의 `.wants/` 또는 관련 디렉터리에 **Symbolic Link를 생성해 자동 활성화 관계를 구성**한다.

개념 예:

```text
/etc/systemd/system/multi-user.target.wants/example.service
        ↓ symbolic link
/usr/lib/systemd/system/example.service
```

그래서 `start`와 `enable`은 내부 목적부터 다르다.

### `start`

```text
지금 Unit 활성화
↓
Unit 정의 확인
↓
ExecStart 실행
```

### `enable`

```text
향후 boot/Target 활성화 시
자동으로 함께 활성화되도록 관계 구성
```

```text
start ≠ enable
```

을 Unit 구조로 설명할 수 있어야 한다.

---

# 10. Unit 파일 위치와 Override

대표 경로:

```text
/usr/lib/systemd/system/
/etc/systemd/system/
/run/systemd/system/
```

## `/usr/lib/systemd/system/`

일반적으로 패키지가 제공하는 기본 Unit 파일이 위치한다.

## `/etc/systemd/system/`

관리자가 만든 Unit 또는 관리자 설정/override가 위치할 수 있다.

패키지가 제공한 `/usr/lib/systemd/system/...` 원본을 직접 수정하면 패키지 업데이트 과정에서 변경이 덮어써질 수 있다. 운영에서는 보통 `/etc/systemd/system/`의 drop-in/override 방식을 우선 고려한다.

> 정확히 어떤 파일과 override가 현재 적용되는지는 `systemctl cat UNIT` 등으로 확인하는 습관이 중요하다.

---

# 11. `systemctl daemon-reload`

## 한 줄 정의

**`systemctl daemon-reload`는 systemd Manager에게 Unit 파일과 관련 설정을 다시 읽으라고 요청하는 명령이다.**

Unit 파일을 수정해도 디스크의 파일만 변경된 상태일 수 있다.

```text
Unit 파일 변경
↓
디스크 내용 변경
↓
systemd가 기존 정의를 이미 로드한 상태일 수 있음
↓
systemctl daemon-reload
↓
Unit 정의 다시 로드
```

### 중요한 구분

```text
daemon-reload
→ Unit 정의 다시 읽기

reload UNIT
→ 실행 중인 Service에 설정 재적용 요청

restart UNIT
→ 실제 Service Process 재시작
```

즉 `daemon-reload`만 실행한다고 Service Process가 자동으로 다시 시작되는 것은 아니다.

---

# 12. 주요 조회 명령

## `systemctl cat UNIT`

Unit 원본과 systemd가 인식하는 drop-in 설정을 함께 확인할 때 유용하다.

```bash
systemctl cat ssh
```

파일 경로 하나만 `cat`하는 것보다 **override를 놓칠 가능성을 줄일 수 있다.**

## `systemctl list-units`

현재 systemd Manager가 로드한 Unit을 런타임 관점에서 조회한다.

## `systemctl list-unit-files`

시스템에 존재하는 Unit 파일과 enable 관련 상태를 조회한다.

```text
list-units
→ 현재 로드된 Unit / 런타임 관점

list-unit-files
→ 설치된 Unit 파일 / enable 상태 관점
```

---

# 13. ⚠️ 헷갈리기 쉬운 부분

> ⚠️ `After=`는 의존성을 자동으로 만드는 설정이 아니다. **순서(Ordering)**와 **활성화 의존성(Requires/Wants)**은 별도로 생각해야 한다.

> ⚠️ `.service` 파일은 실행 프로그램 자체가 아니다. Program을 systemd가 어떻게 관리할지 정의하는 설정이다.

> ⚠️ `ExecStart=`를 수정했다고 systemd가 즉시 새 정의를 사용하는 것은 아니다. Unit 파일 변경 후에는 `daemon-reload` 필요 여부를 검토한다.

> ⚠️ `daemon-reload`는 Service Process 재시작이 아니다.

> ⚠️ `enable`은 현재 서비스 실행과 다른 개념이다. 주로 boot/Target 활성화 관계를 구성한다.

> ⚠️ 패키지 제공 Unit 원본을 직접 수정하면 업데이트 시 덮어써질 수 있다. 관리자 변경은 override 방식을 우선 고려한다.

---

# 14. 💼 실무 운영 흐름

Unit 설정과 관련된 문제라면 다음 순서로 접근하는 습관이 좋다.

```text
증상 확인
↓
systemctl status UNIT
↓
journalctl -u UNIT
↓
systemctl cat UNIT
↓
ExecStart / ExecStartPre / Type / Restart / Dependency 확인
↓
필요한 설정 변경
↓
systemctl daemon-reload
↓
서비스 영향 판단
↓
필요한 경우 reload/restart
↓
systemctl status
↓
journalctl
↓
Process / Port 확인
↓
실제 기능 검증
```

운영자는 단순히 "restart하니 됐다"가 아니라 **어떤 Unit 정의가 어떤 Process를 실행했고, 왜 실패했으며, 변경 후 어떤 근거로 정상이라고 판단했는지** 설명할 수 있어야 한다.

---

# 15. ✅ 핵심 정리

```text
Unit File
├─ [Unit]
│  ├─ Description
│  ├─ Requires / Wants
│  └─ After / Before
│
├─ [Service]
│  ├─ Type
│  ├─ ExecStartPre
│  ├─ ExecStart
│  ├─ ExecStop
│  ├─ ExecReload
│  └─ Restart
│
└─ [Install]
   └─ WantedBy
```

서비스 시작:

```text
systemctl start
→ Unit 정의 확인
→ Dependency / Ordering 처리
→ ExecStartPre
→ ExecStart
→ Process 생성
→ cgroup/상태 추적
→ journal 기록
```

자동 시작 설정:

```text
systemctl enable
→ Target 등과 자동 활성화 관계 구성
→ 일반적으로 Symbolic Link 활용
```

Unit 변경:

```text
Unit 변경
→ daemon-reload
→ 필요하면 reload/restart
→ status
→ journal
→ 실제 기능 검증
```

---

# 16. 🧠 복습 문제

1. Unit 파일과 실행 프로그램은 어떻게 다른가?
2. `[Unit]`, `[Service]`, `[Install]`은 각각 어떤 질문에 답하는 영역인가?
3. `After=database.service`와 `Requires=database.service`의 차이는 무엇인가?
4. `Wants=`와 `Requires=`는 어떤 차이가 있는가?
5. `ExecStartPre=`가 설정 검증에 유용한 이유는 무엇인가?
6. `Type=`은 systemd의 어떤 판단에 영향을 주는가?
7. `Restart=on-failure`가 설정된 Service에 `kill`을 보냈을 때 어떤 일이 발생할 수 있는가?
8. `systemctl enable`과 `systemctl start`가 다른 이유를 Unit/Target 관점에서 설명해보자.
9. `/usr/lib/systemd/system/`의 패키지 원본을 직접 수정하지 않는 것이 좋은 이유는 무엇인가?
10. `daemon-reload`, `reload UNIT`, `restart UNIT`의 차이를 설명해보자.
11. `systemctl cat UNIT`이 단순히 특정 Unit 파일 경로를 `cat`하는 것보다 유용한 이유는 무엇인가?
12. `list-units`와 `list-unit-files`의 차이는 무엇인가?

---

## 다음 실습

SSH 서비스는 현재 원격 접속 경로이므로 우선 읽기 전용으로 Unit 정의를 관찰한다.

```bash
systemctl cat ssh
systemctl show ssh -p FragmentPath -p UnitFileState -p ActiveState -p SubState
```

그 다음에는 SSH가 아닌 **안전한 연습용 Service Unit**을 만들어 다음 흐름을 직접 확인한다.

```text
Unit 작성
→ daemon-reload
→ start
→ status
→ journal
→ stop
→ enable/disable
→ 검증
```
