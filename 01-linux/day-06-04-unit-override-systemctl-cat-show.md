# Day 6-4 — systemd Unit 확인, Override, `systemctl cat/show`

> systemd Service를 운영할 때 단순히 기본 Unit 파일 하나만 보는 것이 아니라, **기본 Unit → Drop-in Override → systemd가 실제로 인식한 Property → 현재 상태 → 로그** 순서로 확인하는 방법을 학습했다.

## 📌 이번에 배운 내용

- Unit 파일 경로와 우선순위
- `/usr/lib/systemd/system/`, `/run/systemd/system/`, `/etc/systemd/system/`
- 패키지 기본 Unit과 관리자 설정의 구분
- Drop-in Override 개념
- `systemctl edit`
- `systemctl cat`
- `systemctl show`
- `FragmentPath`, `UnitFileState`, `ActiveState`, `SubState`
- `systemctl status` / `cat` / `show`의 역할 차이
- Override 시 Directive별 병합/초기화 규칙 주의
- `systemctl daemon-reload`와 Service restart의 차이
- `systemctl list-unit-files --type=service`
- 실제 Ubuntu Server에서 SSH Unit을 읽기 전용으로 관찰

## 📚 목차

1. 왜 Unit 파일 하나만 보면 부족한가
2. Unit 파일 경로와 우선순위
3. Drop-in Override
4. `systemctl edit`
5. `systemctl cat`
6. `systemctl show`
7. 주요 Property 해석
8. `status` / `cat` / `show` 비교
9. Override 주의점
10. `daemon-reload` 다시 정리
11. 🧪 실제 Ubuntu 실습
12. ⚠️ 헷갈리기 쉬운 부분
13. 🔧 Troubleshooting
14. 💼 실무 포인트
15. ✅ 핵심 정리
16. 🧠 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 의미 | 언제 사용하는가 |
|---|---|---|
| `systemctl cat ssh` | 기본 Unit과 Drop-in Override 확인 | 실제 어떤 설정 파일들이 적용 대상인지 볼 때 |
| `systemctl show ssh` | systemd가 인식한 Property 전체 조회 | Unit 내부 상태를 상세히 확인할 때 |
| `systemctl show ssh -p FragmentPath -p UnitFileState -p ActiveState -p SubState` | 필요한 Property만 선택 | 자동화/정밀 확인 |
| `sudo systemctl edit UNIT` | Drop-in Override 편집 | 패키지 원본을 직접 수정하지 않고 관리자 설정 추가 |
| `sudo systemctl daemon-reload` | Unit 정의 재로딩 | Unit/Override 변경 후 systemd가 다시 읽게 할 때 |
| `systemctl list-unit-files --type=service` | Service Unit 파일과 enable 상태 조회 | 설치된 Service Unit 목록을 볼 때 |
| `... | head -20` | 앞 20줄만 출력 | 긴 목록을 빠르게 일부 확인 |

---

# 1. 왜 Unit 파일 하나만 보면 부족한가

systemd Unit은 하나의 파일만 보고 최종 동작이 결정된다고 생각하면 안 된다.

예를 들어 패키지가 제공한 기본 Unit이 다음 위치에 있어도:

```text
/usr/lib/systemd/system/ssh.service
```

관리자가 별도의 Override를 만들 수 있다.

```text
/etc/systemd/system/ssh.service.d/override.conf
```

따라서 운영에서 중요한 질문은 단순히:

```text
"기본 Unit 파일에 뭐라고 적혀 있지?"
```

가 아니라:

```text
"systemd가 최종적으로 어떤 설정을 적용하고 있지?"
```

이다.

---

# 2. Unit 파일 경로와 우선순위

대표적인 systemd Unit 경로:

```text
/etc/systemd/system/
/run/systemd/system/
/usr/lib/systemd/system/
```

일반적인 우선순위는 다음처럼 이해하면 된다.

```text
/etc/systemd/system/
↓ 높은 우선순위
/run/systemd/system/
↓
/usr/lib/systemd/system/
↓ 낮은 우선순위
```

## `/usr/lib/systemd/system/`

주로 패키지가 설치한 기본 Unit 파일이 위치한다.

```text
패키지 제공 기본값
```

Ubuntu 환경에서는 `/lib/systemd/system/`이 보일 수도 있으며 배포판/버전에 따라 경로 표현이 다를 수 있다.

## `/run/systemd/system/`

런타임 중 임시로 생성되는 Unit 관련 설정이 위치할 수 있다.

`/run`은 일반적으로 재부팅 시 초기화되는 런타임 영역이다.

## `/etc/systemd/system/`

관리자가 직접 만든 Unit이나 Override를 두는 대표 위치다.

```text
관리자 정책 / 로컬 변경
```

### 왜 패키지 기본 파일을 직접 수정하지 않는가

```text
/usr/lib/systemd/system/nginx.service 수정
↓
패키지 업데이트
↓
기본 Unit 재설치
↓
관리자 수정 내용이 사라지거나 충돌할 수 있음
```

그래서 운영에서는 패키지 기본값과 관리자 변경을 분리하는 것이 좋다.

---

# 3. Drop-in Override

## 한 줄 정의

**Drop-in Override는 기존 Unit 전체를 복사하지 않고 변경할 설정만 별도 파일에 추가하는 방식이다.**

예:

```text
/usr/lib/systemd/system/ssh.service
        ↓ 기본 Unit

/etc/systemd/system/ssh.service.d/override.conf
        ↓ 관리자 Override
```

예를 들어 `Restart=`만 바꾸고 싶다면:

```ini
[Service]
Restart=on-failure
```

처럼 필요한 부분만 기록할 수 있다.

### 장점

```text
변경 범위가 작음
→ 관리자가 바꾼 내용을 찾기 쉬움
→ 패키지 기본 Unit과 분리됨
→ 업데이트 시 충돌 위험 감소
```

---

# 4. `systemctl edit`

```bash
sudo systemctl edit nginx
```

`systemctl edit`는 Drop-in Override를 만들거나 편집할 때 사용하는 대표 명령이다.

보통 다음과 같은 경로가 사용될 수 있다.

```text
/etc/systemd/system/nginx.service.d/override.conf
```

> SSH는 현재 실습 접속 경로이므로 Day 6-4에서는 실제 SSH Override를 만들지 않고 조회만 했다.

---

# 5. `systemctl cat`

## 한 줄 정의

**`systemctl cat UNIT`은 systemd가 해당 Unit에 사용할 기본 Unit 파일과 Drop-in 파일을 함께 확인하는 명령이다.**

```bash
systemctl cat ssh
```

출력은 환경에 따라 다음처럼 나타날 수 있다.

```text
# /usr/lib/systemd/system/ssh.service
[Unit]
...

[Service]
...
```

Override가 있다면 별도 경로도 이어서 표시될 수 있다.

```text
# /etc/systemd/system/ssh.service.d/override.conf
...
```

### 왜 `cat /usr/lib/...`보다 유용한가

```text
cat 특정 파일
→ 그 파일 하나만 확인

systemctl cat UNIT
→ Unit 기본 정의 + Drop-in 확인
```

따라서 "현재 이 Unit에 어떤 설정 파일이 관련되어 있는가"를 볼 때 더 안전하다.

---

# 6. `systemctl show`

## 한 줄 정의

**`systemctl show UNIT`은 systemd가 현재 해당 Unit에 대해 인식하고 있는 상세 Property를 출력한다.**

```bash
systemctl show ssh
```

출력량이 많기 때문에 필요한 Property만 `-p`로 선택할 수 있다.

```bash
systemctl show ssh \
  -p FragmentPath \
  -p UnitFileState \
  -p ActiveState \
  -p SubState
```

## `-p`

```text
-p = property
```

특정 속성만 출력하도록 필터링한다.

---

# 7. 주요 Property 해석

## `FragmentPath`

예:

```text
FragmentPath=/usr/lib/systemd/system/ssh.service
```

systemd가 해당 Unit의 주된 Unit 파일로 읽은 경로를 의미한다.

## `UnitFileState`

예:

```text
UnitFileState=enabled
```

Unit 파일의 enable 상태를 보여준다.

```text
systemctl is-enabled ssh
```

에서 확인하던 개념과 연결된다.

## `ActiveState`

예:

```text
ActiveState=active
```

Unit의 큰 범주 상태다.

대표적인 값:

```text
active
inactive
failed
activating
deactivating
```

## `SubState`

예:

```text
SubState=running
```

`ActiveState`보다 세부적인 Unit Type별 상태다.

예:

```text
ActiveState=active
SubState=running
```

은 `systemctl status`에서 다음처럼 보일 수 있다.

```text
active (running)
```

즉:

```text
active  → ActiveState
running → SubState
```

라고 연결할 수 있다.

---

# 8. `status` / `cat` / `show` 비교

| 명령 | 중심 관점 | 주요 목적 |
|---|---|---|
| `systemctl status UNIT` | 사람이 읽기 좋은 현재 상태 | Active, PID, Resource, 최근 로그 빠른 확인 |
| `systemctl cat UNIT` | 설정 파일 | 기본 Unit과 Override 확인 |
| `systemctl show UNIT` | systemd 내부 Property | systemd가 실제 인식한 상세 값 확인 |

운영에서는 다음처럼 사용한다.

```text
서비스가 이상하다
↓
status
→ 현재 증상 확인

설정이 어디서 왔는지 궁금하다
↓
cat

systemd가 어떤 값으로 인식했는지 정확히 보고 싶다
↓
show
```

---

# 9. Override 주의점

Override는 단순히 "아래 파일이 위 파일의 같은 줄을 무조건 덮어쓴다"라고 이해하면 부족하다.

systemd Directive는 종류에 따라 병합 또는 초기화 규칙이 다를 수 있다.

특히:

```text
ExecStart=
Environment=
After=
Wants=
```

등은 Directive 특성을 확인해야 한다.

예를 들어 `ExecStart=`를 교체해야 하는 상황에서는 다음처럼 기존 값을 먼저 비우는 형태가 필요한 경우가 있다.

```ini
[Service]
ExecStart=
ExecStart=/new/path/program
```

첫 번째 빈 `ExecStart=`는 기존 값을 reset하는 의미로 사용될 수 있다.

> ⚠️ Override는 단순 문자열 덮어쓰기가 아니다. 실제 변경 전에는 해당 Directive의 systemd 문서와 현재 `systemctl cat/show` 결과를 확인한다.

---

# 10. `daemon-reload` 다시 정리

Unit 파일 또는 Drop-in을 변경하면:

```bash
sudo systemctl daemon-reload
```

를 사용해 systemd Manager가 Unit 정의를 다시 읽게 한다.

```text
파일 수정
↓
Disk의 내용 변경
↓
systemd는 이전 정의를 이미 읽은 상태일 수 있음
↓
daemon-reload
↓
Unit 정의 다시 로딩
```

하지만:

```text
daemon-reload
≠ Service Process restart
```

이다.

`daemon-reload`는 Unit 정의를 다시 읽는 작업이고, 실제 실행 중인 Service Process에 새 설정을 반영하려면 Service 특성에 따라 reload/restart 등의 별도 조치가 필요할 수 있다.

---

# 11. 🧪 실제 Ubuntu 실습

이번 실습은 SSH로 서버에 접속한 상태이므로 **서비스를 수정하거나 중지하지 않고 읽기 전용 조회만 수행**했다.

## 11-1. SSH Unit 파일 확인

```bash
systemctl cat ssh
```

### 확인 목표

```text
ssh.service 기본 Unit 파일 위치
[Unit] / [Service] / [Install] 내용
Drop-in Override 존재 여부
```

실제 출력은 환경/Ubuntu 버전에 따라 다를 수 있으므로 문서에는 특정 결과를 추정해서 기록하지 않는다.

## 11-2. systemd가 인식한 Property 확인

```bash
systemctl show ssh \
  -p FragmentPath \
  -p UnitFileState \
  -p ActiveState \
  -p SubState
```

### 확인 목표

```text
FragmentPath  → 주 Unit 파일 경로
UnitFileState → enable 상태
ActiveState   → 큰 범주 상태
SubState      → 세부 상태
```

## 11-3. 설치된 Service Unit 목록 확인

```bash
systemctl list-unit-files --type=service | head -20
```

분해:

```text
systemctl list-unit-files
→ 설치된 Unit 파일과 상태를 조회

--type=service
→ Service Unit만 필터링

|
→ 앞 명령 stdout을 다음 명령 stdin으로 전달

head -20
→ 앞 20줄만 출력
```

### `list-units`와 `list-unit-files` 비교

```text
systemctl list-units
→ 현재 systemd에 로드된 Unit 중심

systemctl list-unit-files
→ 설치된 Unit 파일과 enable 상태 중심
```

둘은 같은 목록을 보여주는 명령이 아니다.

---

# 12. ⚠️ 헷갈리기 쉬운 부분

## `/usr/lib/systemd/system/`이 항상 최종 설정은 아니다

> ⚠️ 처음에는 기본 Unit 파일 하나만 보면 된다고 생각하기 쉽다. 실제로는 `/etc/systemd/system/`의 Override나 다른 설정이 더 높은 우선순위로 적용될 수 있다.

## `systemctl cat`과 `systemctl show`

```text
cat  → 설정 파일 관점
show → systemd가 인식한 Property 관점
```

## `ActiveState`와 `SubState`

```text
ActiveState → 큰 상태
SubState    → 세부 상태
```

## `daemon-reload`와 `restart`

```text
daemon-reload → Unit 정의 다시 읽기
restart       → Service Process 재시작
```

## `list-units`와 `list-unit-files`

```text
list-units      → 로드된 Unit 관점
list-unit-files → 설치된 Unit 파일/enable 상태 관점
```

---

# 13. 🔧 Troubleshooting

## 증상

```text
"Unit 파일을 수정했는데 서비스 동작이 예상과 다르다."
```

## 접근

```text
1. systemctl cat UNIT
   → 기본 Unit / Override 확인

2. systemctl show UNIT
   → systemd가 인식한 Property 확인

3. 변경 직후라면 daemon-reload 여부 확인

4. systemctl status UNIT
   → 현재 runtime 상태 확인

5. journalctl -u UNIT
   → 시작/실패 로그 확인

6. 필요하면 실제 Process / Port / 기능 검증
```

### 가능한 원인

```text
다른 Drop-in Override 존재
Unit 파일 수정 후 daemon-reload 누락
잘못된 Unit 파일을 수정
Directive 병합 규칙 오해
Service restart/reload가 필요한데 수행하지 않음
```

---

# 14. 💼 실무 포인트

Service 설정 조사 시 다음 순서를 습관화한다.

```text
status
→ cat
→ show
→ journal
```

설정 변경 시에는:

```text
현재 상태 확인
→ 기본 Unit / Override 확인
→ 변경 범위 최소화
→ daemon-reload
→ 영향 판단
→ 필요한 reload/restart
→ status
→ journal
→ 실제 기능 검증
```

특히 패키지 기본 Unit 파일을 직접 수정하기보다 Drop-in Override를 사용하는 이유를 이해하는 것이 중요하다.

---

# 15. ✅ 핵심 정리

```text
/usr/lib/systemd/system/
→ 패키지 기본 Unit

/etc/systemd/system/
→ 관리자 Unit / Override

Drop-in Override
→ 필요한 설정만 별도 파일로 변경

systemctl cat
→ 기본 Unit + Override 확인

systemctl show
→ systemd가 인식한 Property 확인

FragmentPath
→ 주 Unit 파일 경로

UnitFileState
→ enable 상태

ActiveState
→ 큰 상태

SubState
→ 세부 상태

daemon-reload
→ Unit 정의 다시 읽기
```

가장 중요한 운영 관점:

```text
"파일에 뭐라고 적혀 있나?"
만 보지 말고
↓
"systemd가 최종적으로 무엇을 인식하고 있나?"
까지 확인한다.
```

---

# 16. 🧠 복습 문제

1. `/usr/lib/systemd/system/`과 `/etc/systemd/system/`의 역할 차이는 무엇인가?
2. 패키지가 제공한 Unit 파일을 직접 수정하는 것이 위험한 이유는 무엇인가?
3. Drop-in Override란 무엇인가?
4. `systemctl edit`는 어떤 목적으로 사용하는가?
5. `systemctl cat`과 일반 `cat /path/to/unit`의 차이는 무엇인가?
6. `systemctl show`는 무엇을 보여주는가?
7. `FragmentPath`는 무엇을 의미하는가?
8. `UnitFileState=enabled`는 어떤 상태인가?
9. `ActiveState=active`, `SubState=running`은 `systemctl status`에서 어떻게 표현될 수 있는가?
10. `systemctl status`, `cat`, `show`의 역할을 각각 설명해보자.
11. Unit 파일을 수정한 뒤 `daemon-reload`가 필요한 이유는 무엇인가?
12. `daemon-reload`가 실행 중 Service Process를 자동으로 재시작하는가?
13. Override의 `ExecStart=`를 변경할 때 단순히 한 줄 추가하는 것으로 끝나지 않을 수 있는 이유는 무엇인가?
14. `list-units`와 `list-unit-files`의 차이는 무엇인가?
15. "설정을 수정했는데 반영되지 않는다"는 장애에서 어떤 순서로 조사할 것인가?

---

## 다음 학습

다음은 **안전한 연습용 systemd Service Unit을 직접 생성**한다.

목표:

```text
Unit 파일 작성
→ daemon-reload
→ start
→ status
→ journal
→ stop
→ enable / disable
→ 일부러 오류 발생
→ Troubleshooting
```

실제 SSH 같은 중요 서비스가 아니라 학습용 Service를 사용해 Service Lifecycle 전체를 안전하게 실습한다.
