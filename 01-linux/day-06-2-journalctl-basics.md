# Day 6-2 — systemd Journal과 `journalctl` 기초

> `systemctl status`에서 보이는 최근 로그가 어디서 오는지 이해하고, systemd 환경에서 로그를 중앙집중식으로 수집·조회하는 **journal / systemd-journald / journalctl**의 관계를 학습했다. 아직 실제 명령 실행 전 단계이며, 이번 문서는 로그 조회에 들어가기 전에 필요한 개념과 옵션 의미를 정리한다.

## 📌 이번에 배운 내용

- systemd journal이 무엇인지와 왜 필요한지
- `systemd-journald` daemon의 역할
- `systemctl`과 `journalctl`의 차이
- `systemctl status`의 로그만으로 장애 분석이 부족할 수 있는 이유
- `journalctl -u`, `-n`, `-f`, `-b`, `--since`, `--until`, `--no-pager`, `-p`
- 현재 부팅과 이전 부팅 로그 조회 개념
- volatile / persistent journal 차이
- journal의 구조화된 로그와 metadata
- 로그 Priority(severity) 0~7
- 서비스 장애 시 로그 범위를 좁혀가는 실무 흐름

## 📚 목차

1. systemd journal이란?
2. 왜 journal이 필요한가?
3. `systemd-journald`란?
4. `journalctl`이란?
5. `systemctl`과 `journalctl` 비교
6. journal 로그의 수집 구조
7. 구조화된 로그와 Metadata
8. `journalctl` 주요 옵션
9. Boot 기준 로그 조회
10. Volatile / Persistent Journal
11. Log Priority
12. 로그 한 줄 읽는 법
13. Troubleshooting에서의 사용 흐름
14. 헷갈리기 쉬운 부분
15. 실무 포인트
16. 핵심 정리
17. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 의미 | 언제 사용하는가 |
|---|---|---|
| `journalctl -u ssh` | `ssh` Unit 로그 조회 | 특정 서비스 로그만 보고 싶을 때 |
| `journalctl -u ssh -n 20` | SSH 최근 20개 로그 조회 | 최근 이벤트부터 빠르게 확인할 때 |
| `journalctl -u ssh -n 20 --no-pager` | pager 없이 최근 로그 출력 | 복사, 자동화, 짧은 로그 확인 |
| `journalctl -u ssh -f` | SSH 로그 실시간 추적 | 접속/재시작 등 이벤트를 실시간 관찰할 때 |
| `journalctl -u ssh --since "10 minutes ago"` | 최근 10분 로그 | 장애 발생 시각 주변으로 범위 축소 |
| `journalctl -u ssh --since "..." --until "..."` | 특정 시간 구간 로그 | 장애 시간대만 정밀하게 조사 |
| `journalctl -b` | 이번 부팅의 로그 | 재부팅 이후 문제 분석 |
| `journalctl -b -1` | 이전 부팅의 로그 | 이전 부팅에서 발생한 장애 확인 |
| `journalctl -u ssh -b` | 이번 부팅의 SSH 로그 | Boot + Unit 조건을 동시에 적용 |
| `journalctl -p err` | error 이상 Priority 로그 | 심각한 로그부터 좁혀 볼 때 |

---

# 1. systemd journal이란?

## 한 줄 정의

**systemd journal은 systemd 환경에서 다양한 시스템·서비스 로그를 수집하고 저장한 로그 데이터 집합이다.**

`systemctl status ssh` 아래에서 다음과 같은 내용을 이미 확인했다.

```text
systemd[1]: Starting ssh.service ...
sshd[1040]: Server listening on 0.0.0.0 port 22.
systemd[1]: Started ssh.service ...
```

이 최근 로그 일부도 journal에서 가져온 정보다.

핵심 구조는 다음과 같다.

```text
서비스 / 프로세스 / Kernel 등
↓
로그 발생
↓
systemd-journald가 수집
↓
systemd journal에 저장
↓
journalctl로 조회
```

---

# 2. 왜 journal이 필요한가?

Linux 시스템에는 로그가 여러 위치에 존재할 수 있다.

예:

```text
/var/log/auth.log
/var/log/syslog
/var/log/nginx/error.log
/var/log/mysql/error.log
```

서비스마다 로그 위치와 형식이 다르면 장애 상황에서 먼저 "로그가 어디 있는가"부터 찾아야 한다. systemd journal은 여러 출처의 로그를 중앙에서 수집하고, **Unit·시간·Boot·Priority 같은 조건으로 조회**할 수 있게 한다.

```text
로그가 많음
↓
어느 서비스인가?
↓
언제 발생했는가?
↓
어느 부팅에서 발생했는가?
↓
얼마나 심각한 로그인가?
↓
조건을 조합해서 범위를 축소
```

이것이 journal이 운영에서 유용한 이유다.

> journal이 존재한다고 해서 `/var/log`의 개별 로그 파일이 항상 사라지는 것은 아니다. 서비스와 배포판 설정에 따라 전통적인 파일 로그와 journal이 함께 사용될 수 있다.

---

# 3. `systemd-journald`란?

## 한 줄 정의

**`systemd-journald`는 시스템에서 발생하는 여러 로그를 수집하고 journal에 기록하는 daemon이다.**

앞에서 배운 Service / Daemon 개념과 연결하면:

```text
systemd-journald
→ 실제 로그 수집 daemon / process

systemd-journald.service
→ systemd가 해당 기능을 관리하는 Service Unit
```

따라서 `journal`은 조회 명령의 이름이 아니라 로그 저장 체계이고, `systemd-journald`가 수집 역할을 담당하며, `journalctl`이 조회 도구다.

```text
systemd-journald → 수집
journal          → 저장된 로그 데이터
journalctl       → 조회
```

---

# 4. `journalctl`이란?

## 한 줄 정의

**`journalctl`은 systemd journal에 저장된 로그를 조회하고 필터링하는 명령어다.**

`journalctl`을 옵션 없이 실행하면 접근 가능한 journal 로그를 시간 순서로 폭넓게 조회한다. 실제 서버에서는 로그 양이 많을 수 있으므로 보통 Unit, 시간, Boot, Priority 등으로 범위를 좁혀 사용한다.

예:

```bash
journalctl -u ssh
```

```text
journalctl
→ journal 조회 도구

-u ssh
→ ssh Unit으로 범위를 제한
```

---

# 5. `systemctl`과 `journalctl` 비교

두 명령은 서로 관련 있지만 목적이 다르다.

| 명령 | 주 목적 | 대표 질문 |
|---|---|---|
| `systemctl` | Unit 상태 확인 및 제어 | "서비스가 지금 어떤 상태인가?" |
| `journalctl` | 로그 조회 및 필터링 | "서비스가 왜 이런 상태가 되었는가?" |

예:

```bash
systemctl status ssh
```

확인하는 것:

```text
Loaded
Active
Main PID
CGroup
최근 로그 일부
```

반면:

```bash
journalctl -u ssh
```

은 SSH Unit과 관련된 더 넓은 범위의 journal 기록을 확인하는 데 사용한다.

실무에서는 둘 중 하나만 쓰는 것이 아니라 연결해서 본다.

```text
systemctl status
→ 현재 상태와 최근 단서 확인
→ journalctl -u
→ 더 넓은 로그에서 원인 추적
```

---

# 6. journal 로그의 수집 구조

개념을 단순화하면 다음과 같다.

```text
Application / Service
        │
        ├─ stdout
        ├─ stderr
        ├─ syslog 계열
        └─ 기타 systemd가 수집 가능한 로그
        ↓
systemd-journald
        ↓
Journal
        ↓
journalctl
```

Kernel 메시지 등도 journal에서 조회할 수 있다. 다만 모든 애플리케이션 로그가 무조건 journal로 들어온다고 단정하면 안 된다. 애플리케이션이 자체 파일에만 기록하도록 설정되어 있을 수도 있기 때문이다.

---

# 7. 구조화된 로그와 Metadata

journal의 중요한 특징 중 하나는 로그 메시지뿐 아니라 여러 **metadata**를 함께 관리할 수 있다는 점이다.

Metadata는 "데이터에 대한 데이터"라는 뜻이다. 로그 본문이 무엇인지뿐 아니라 **누가, 언제, 어디서 남겼는지 설명하는 부가 정보**라고 생각하면 된다.

예:

```text
Timestamp
Hostname
Process Name
PID
UID
Unit Name
Priority
Message
```

따라서 단순한 텍스트:

```text
Server started
```

보다 다음과 같은 질문에 답하기 쉬워진다.

```text
언제 발생했는가?
어느 서버인가?
어떤 Process인가?
PID는 무엇인가?
어느 Unit에 속하는가?
Priority는 무엇인가?
실제 Message는 무엇인가?
```

이 metadata 덕분에 `journalctl -u`, `-b`, `-p` 같은 필터링이 가능하다.

---

# 8. `journalctl` 주요 옵션

## 8-1. `-u` — Unit 선택

```bash
journalctl -u ssh
```

`-u`는 **Unit**을 기준으로 필터링한다.

```text
전체 journal
↓
-u ssh
↓
ssh.service 관련 로그
```

서비스 장애를 조사할 때 가장 자주 사용할 조건 중 하나다.

---

## 8-2. `-n` — 최근 로그 개수 제한

```bash
journalctl -u ssh -n 20
```

`-n 20`은 최근 로그 20개만 보여달라는 의미다.

로그가 매우 많을 때 처음부터 전부 읽지 않고 최근 이벤트부터 빠르게 확인할 수 있다.

```text
전체 SSH 로그
↓
최근 20개만
```

---

## 8-3. `--no-pager` — Pager 사용하지 않기

```bash
journalctl -u ssh -n 20 --no-pager
```

출력이 길면 `journalctl`은 환경에 따라 `less` 같은 pager를 사용할 수 있다.

`--no-pager`는 pager를 거치지 않고 결과를 터미널에 바로 출력한다.

활용 예:

```text
짧은 결과 확인
복사/붙여넣기
스크립트 또는 파이프라인에서 사용
```

---

## 8-4. `-f` — Follow

```bash
journalctl -u ssh -f
```

`-f`는 **follow**의 의미로 새 로그가 발생하면 계속 이어서 출력한다.

Day 1에서 배운:

```bash
tail -f FILE
```

과 비슷한 사용 목적이다.

예:

```text
Terminal A
journalctl -u ssh -f

Terminal B
새 SSH 접속 시도

Terminal A
→ 새로운 인증/세션 로그가 실시간으로 나타남
```

`Ctrl+C`로 실시간 추적을 종료할 수 있다.

---

## 8-5. `--since` — 시작 시간 지정

```bash
journalctl -u ssh --since "10 minutes ago"
```

지정한 시각 이후의 로그만 조회한다.

절대 시각도 사용할 수 있다.

```bash
journalctl -u ssh --since "2026-09-08 04:00:00"
```

장애 접수 시간이 알려져 있다면 매우 유용하다.

```text
"04:05쯤부터 SSH가 안 됩니다"
↓
04:00 전후로 로그 범위를 제한
↓
장애 직전과 직후 이벤트 확인
```

---

## 8-6. `--until` — 종료 시간 지정

```bash
journalctl -u ssh --until "2026-09-08 04:20:00"
```

지정한 시각까지의 로그만 조회한다.

`--since`와 조합하면 특정 시간 구간을 만들 수 있다.

```bash
journalctl -u ssh \
  --since "2026-09-08 04:00:00" \
  --until "2026-09-08 04:20:00"
```

즉:

```text
Unit 조건
+
시작 시각
+
종료 시각
=
장애 시간대의 해당 서비스 로그
```

---

# 9. Boot 기준 로그 조회

## `-b`

```bash
journalctl -b
```

`-b`는 **boot** 기준으로 로그 범위를 선택할 때 사용한다.

옵션 인자를 생략하면 일반적으로 현재 Boot의 로그를 본다.

```text
-b 또는 -b 0
→ 현재 부팅

-b -1
→ 바로 이전 부팅

-b -2
→ 그 이전 부팅
```

예:

```bash
journalctl -u ssh -b
```

의미:

```text
-u ssh
→ SSH Unit만

-b
→ 이번 부팅만
```

즉 여러 필터 조건을 조합한 것이다.

## 왜 Boot 기준이 중요한가?

서버가 재부팅된 후 문제가 생겼다고 하자.

```text
현재 시스템은 정상
하지만 재부팅 직전 왜 장애가 났는지 확인 필요
```

이때 과거 Boot journal이 남아 있다면:

```bash
journalctl -b -1
```

로 이전 부팅 기록을 조사할 수 있다.

---

# 10. Volatile / Persistent Journal

journal이 항상 재부팅 후에도 남아 있는 것은 아니다.

## Volatile Journal

**휘발성 저장 방식**으로 런타임 파일시스템 영역에 저장된다.

대표적인 경로:

```text
/run/log/journal/
```

`/run`은 일반적으로 재부팅 시 새로 만들어지는 런타임 영역이므로 과거 로그가 유지되지 않을 수 있다.

## Persistent Journal

**디스크에 지속적으로 저장하는 방식**이다.

대표적인 경로:

```text
/var/log/journal/
```

이 경우 설정과 보존 정책이 허용하는 범위에서 재부팅 전 로그도 남길 수 있다.

### 중요한 결론

```text
journalctl -b -1
```

명령 자체가 존재한다고 해서 반드시 이전 Boot 로그가 있는 것은 아니다.

```text
이전 Boot 로그를 조회하려면
→ 그 로그가 실제로 보존되어 있어야 함
```

이것이 "명령이 지원된다"와 "데이터가 실제 존재한다"를 구분해야 하는 이유다.

---

# 11. Log Priority

Linux/syslog 계열 로그에는 심각도를 나타내는 **Priority / Severity Level**이 있다.

| 숫자 | 이름 | 의미 |
|---:|---|---|
| 0 | `emerg` | 시스템을 사용할 수 없는 매우 심각한 상황 |
| 1 | `alert` | 즉시 조치가 필요한 상황 |
| 2 | `crit` | 치명적인 오류 |
| 3 | `err` | 오류 |
| 4 | `warning` | 경고 |
| 5 | `notice` | 주목할 만한 정상 이벤트 |
| 6 | `info` | 일반 정보 |
| 7 | `debug` | 디버깅 정보 |

**숫자가 작을수록 더 심각하다.**

예:

```bash
journalctl -p err
```

은 error 수준과 그보다 심각한 Priority를 조회하는 용도로 사용할 수 있다.

즉 대략:

```text
0 emerg
1 alert
2 crit
3 err
```

범위를 포함한다.

> `-p err`를 "err만 딱 한 종류"라고 생각하지 않는다. Priority 범위 규칙 때문에 더 심각한 레벨도 함께 포함될 수 있다.

---

# 12. 로그 한 줄 읽는 법

이전에 SSH status에서 본 예:

```text
Sep 08 04:10:55 ubuntu-server sshd[1040]: Server listening on 0.0.0.0 port 22.
```

기본적으로 다음처럼 읽을 수 있다.

| 부분 | 의미 |
|---|---|
| `Sep 08 04:10:55` | 로그 시각 |
| `ubuntu-server` | Hostname |
| `sshd` | 로그를 남긴 Process/프로그램 이름 |
| `[1040]` | PID |
| `Server listening ...` | 실제 Message |

하지만 journal 내부에는 화면에 보이는 이 문자열 외에도 Unit, UID, Priority 등 더 많은 metadata가 존재할 수 있다.

---

# 13. 🔧 Troubleshooting에서의 사용 흐름

서비스 장애를 만났을 때 처음부터 전체 로그를 무작정 읽지 않는다.

예를 들어 Nginx가 동작하지 않는다고 가정한다.

```text
증상
"웹 접속 불가"
↓
systemctl status nginx
↓
현재 상태 / 실패 시각 / 최근 로그 확인
↓
journalctl -u nginx -n 50
↓
해당 Unit의 최근 로그 확인
↓
장애 시각이 알려졌다면 --since / --until
↓
필요하면 -p err 등으로 심각도 축소
↓
Config / Permission / Port / Process 확인
↓
원인 판단
↓
안전한 조치
↓
상태 및 로그 재확인
↓
실제 HTTP 요청으로 기능 검증
```

핵심은 **필터를 많이 쓰는 것 자체가 목적이 아니라, 장애 범위를 논리적으로 좁히는 것**이다.

### 가설별 로그 단서 예

```text
설정 문법 오류
→ syntax / configuration 관련 message

권한 문제
→ permission denied

포트 충돌
→ address already in use / bind 실패

의존성 문제
→ dependency failed

인증 문제
→ authentication failed / denied 계열 message
```

로그 메시지는 원인을 증명하는 자료가 될 수 있지만, 로그 한 줄만 보고 성급하게 결론 내리지 않는다. Process, Port, Config, Permission 등 다른 상태와 교차 검증한다.

---

# 14. ⚠️ 헷갈리기 쉬운 부분

## journal과 `journalctl`

> ⚠️ 처음에는 둘을 같은 것으로 생각하기 쉽다. **journal은 저장된 로그 체계/데이터이고, `journalctl`은 그것을 조회하는 명령어**다.

```text
journal    → 로그 데이터
journalctl → 조회 도구
```

## `systemd-journald`와 `journalctl`

```text
systemd-journald → 로그 수집 daemon
journalctl       → 로그 조회 도구
```

둘은 역할이 다르다.

## `systemctl status`와 `journalctl`

> ⚠️ `systemctl status` 아래에 로그가 보인다고 해서 전체 journal을 보여주는 것은 아니다. 상태 확인을 돕기 위한 최근 로그 일부가 표시되는 것이다.

## `-f`

> ⚠️ `-f`는 기존 로그를 한 번 출력하고 끝내는 옵션이 아니라 **새 로그를 계속 Follow**한다. 종료하려면 일반적으로 `Ctrl+C`를 사용한다.

## `-b -1`

> ⚠️ 이전 Boot를 선택하는 방법이지 과거 로그를 만들어내는 명령이 아니다. 이전 Boot journal이 보존되어 있지 않으면 조회할 데이터가 없다.

## Journal 저장 방식

> ⚠️ systemd journal은 단순 `.log` 텍스트 파일 하나로만 저장되는 구조가 아니다. 구조화된 journal 형식을 사용하므로 일반적으로 `journalctl`로 조회한다.

## Priority 숫자

> ⚠️ 숫자가 클수록 심각한 것이 아니다. **0이 가장 심각하고 7이 debug**다.

---

# 15. 💼 실무 포인트

## 1. 로그는 시간 범위를 먼저 맞춘다

사용자 신고 시각과 서버 로그의 timezone이 다르면 엉뚱한 구간을 분석할 수 있다. 이전 SSH 실습에서 서버가 UTC를 사용하고 있었으므로 장애 시각을 비교할 때 timezone 확인이 중요하다.

```text
사용자 신고 시간
↓
서버 timezone 확인
↓
동일 기준으로 시간 변환
↓
--since / --until 범위 설정
```

## 2. 전체 로그보다 범위를 좁혀 본다

```text
전체 시스템 로그
→ Unit
→ Boot
→ 시간
→ Priority
```

처럼 범위를 좁혀야 중요한 단서를 찾기 쉽다.

## 3. 로그만 보고 장애 해결 완료라고 하지 않는다

예를 들어 로그에 `Started`가 있어도 실제 서비스 기능이 정상이라는 보장은 없다.

```text
로그 확인
→ Process
→ Port
→ Config
→ 실제 요청/접속
```

까지 검증한다.

## 4. 재부팅 전 로그 보존 정책도 운영 요소다

장애 후 서버가 재부팅되었는데 journal이 volatile이었다면 중요한 원인 로그가 사라질 수 있다. 운영 환경에서는 로그 보존 기간, 저장 위치, 용량 정책, 중앙 로그 수집까지 별도로 설계하는 이유다.

---

# 16. ✅ 핵심 정리

```text
systemd-journald
→ 로그를 수집하는 daemon

journal
→ 수집된 구조화 로그 데이터

journalctl
→ journal을 조회하는 명령
```

가장 기본적인 조회 흐름:

```text
systemctl status SERVICE
↓
journalctl -u SERVICE -n 50
↓
--since / --until로 장애 시간 범위 지정
↓
필요하면 -b / -p 등 추가 필터
↓
로그 단서와 Process / Port / Config / Permission 교차 확인
↓
조치
↓
로그 + 실제 기능 재검증
```

주요 옵션:

```text
-u UNIT       → 특정 Unit
-n N          → 최근 N개
-f            → 실시간 Follow
-b            → Boot 기준
--since       → 시작 시간
--until       → 종료 시간
--no-pager    → pager 없이 출력
-p PRIORITY   → 로그 Priority 기준
```

저장 방식:

```text
/run/log/journal  → volatile 가능
/var/log/journal  → persistent 가능
```

그리고 가장 중요한 운영 원칙:

> **로그는 정답 그 자체가 아니라 원인을 좁히는 증거다. 상태·프로세스·포트·설정·권한·실제 기능과 함께 검증한다.**

---

# 17. 🧠 복습 문제

1. systemd journal과 `journalctl`은 어떻게 다른가?
2. `systemd-journald`는 Service / Daemon 관점에서 어떤 역할을 하는가?
3. `systemctl status`에 로그가 보이는데도 `journalctl`이 필요한 이유는 무엇인가?
4. `journalctl -u ssh`에서 `-u`는 무엇을 기준으로 필터링하는가?
5. `-n 20`과 `-f`의 차이는 무엇인가?
6. `--since`와 `--until`을 함께 사용하는 이유는 무엇인가?
7. `journalctl -b`와 `journalctl -b -1`의 차이는 무엇인가?
8. `journalctl -b -1`을 실행해도 이전 부팅 로그가 없을 수 있는 이유는 무엇인가?
9. volatile journal과 persistent journal의 차이는 무엇인가?
10. journal을 "구조화된 로그"라고 부르는 이유는 무엇인가?
11. Priority에서 `0`과 `7` 중 어느 쪽이 더 심각한가?
12. `journalctl -p err`가 장애 분석에서 유용한 이유는 무엇인가?
13. 서비스 장애에서 로그만 보고 원인을 확정하면 안 되는 이유는 무엇인가?
14. 장애 신고 시각과 서버 timezone을 맞춰야 하는 이유는 무엇인가?

---

## 다음 실습

개념 학습 후 실제 Ubuntu Server에서 다음 명령으로 journal을 관찰한다.

```bash
journalctl -u ssh -n 20 --no-pager
journalctl -u ssh -b -n 20 --no-pager
```

실습에서는 **로그 시각 → hostname → process/PID → message → Unit 상태와의 연결** 순서로 읽고, `systemctl status ssh`에서 보았던 로그와 journal의 관계를 확인한다.
