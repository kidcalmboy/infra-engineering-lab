# Day 7-4 — Package 업데이트, Version, upgrade/full-upgrade, hold/unhold

> 설치된 Package와 Repository의 Candidate Version을 비교하고, 업데이트 가능한 Package를 확인한 뒤 `upgrade`, `full-upgrade`, `hold/unhold`을 통해 변경 범위를 통제하는 방법을 정리한다.

## 📌 이번에 배운 내용

- `apt list --upgradable`
- `apt policy PACKAGE`
- Installed / Candidate의 의미
- Debian/Ubuntu Version 문자열의 기본 구조
- Epoch / Upstream Version / Debian·Ubuntu Revision
- `apt upgrade`
- `apt full-upgrade`
- `upgrade`와 `full-upgrade` 차이
- `--dry-run`으로 업데이트 영향 미리보기
- 특정 Package만 업그레이드하는 `--only-upgrade`
- `apt-mark hold`, `unhold`, `showhold`
- 특정 Version 지정 설치
- Service Package 업데이트 후 검증
- Kernel Package 업데이트와 재부팅
- Package 파일 교체와 실행 중 Process의 관계
- 운영 서버에서 안전하게 업데이트하는 흐름

## 📚 목차

1. Package 업데이트 전체 흐름
2. 업데이트 가능한 Package 확인
3. Installed / Candidate
4. Version 문자열 구조
5. `apt upgrade`
6. `apt full-upgrade`
7. `upgrade` vs `full-upgrade`
8. `--dry-run`
9. 특정 Package만 업그레이드
10. hold / unhold
11. 특정 Version 설치
12. Service Package 업데이트
13. Kernel 업데이트와 Reboot
14. Package 파일과 실행 중 Process
15. 운영 서버 업데이트 절차
16. Troubleshooting 시작점
17. 실습
18. 핵심 정리
19. 복습 문제

---

# 1. Package 업데이트 전체 흐름

업데이트는 다음 구조로 이해하면 된다.

```text
Repository
↓
apt update
↓
Local Package Index 최신화
↓
Installed Version vs Candidate Version 비교
↓
apt list --upgradable
↓
변경 영향 확인
↓
upgrade / full-upgrade 판단
↓
필요하면 hold 적용
↓
실제 업데이트
↓
Service / Process / Log / 기능 검증
```

핵심:

> **업데이트는 명령 실행보다 먼저 "무엇이 바뀌는지 확인하는 과정"이 중요하다.**

---

# 2. 업데이트 가능한 Package 확인

```bash
apt list --upgradable
```

## 한 줄 정의

**현재 설치된 Package 중 더 높은 Candidate Version이 존재하는 Package 목록을 보여준다.**

개념적으로:

```text
Installed Version
<
Candidate Version
```

인 경우 업그레이드 대상이 될 수 있다.

### 왜 먼저 확인하는가?

운영 서버에서 바로 `apt upgrade`를 실행하면 변경되는 Package가 많을 수 있다.

먼저 목록을 확인하면:

```text
어떤 Package가 바뀌는가?
중요 Service Package가 포함되는가?
Kernel이 포함되는가?
예상치 못한 Runtime/Library가 포함되는가?
```

를 판단할 수 있다.

---

# 3. `apt policy PACKAGE`

예:

```bash
apt policy openssh-client
```

주요 항목:

```text
Installed
→ 현재 시스템에 설치된 Version

Candidate
→ 현재 APT가 설치/업데이트 대상으로 선택하는 Version

Version table
→ Repository에서 확인 가능한 Version과 우선순위 정보
```

예시 개념:

```text
Installed: 1.0
Candidate: 1.1
```

이면 업데이트 가능한 상태다.

반대로:

```text
Installed: 1.1
Candidate: 1.1
```

이면 일반적으로 현재 Candidate와 동일하다.

> 실제 Version 값은 Repository와 Ubuntu Release에 따라 달라진다.

---

# 4. Debian/Ubuntu Version 문자열

Linux Package Version은 단순히 `1.2.3`만 있는 것이 아니다.

예:

```text
1:9.6p1-3ubuntu13.5
```

개념적으로:

```text
1:
→ Epoch

9.6p1
→ Upstream Version

-3ubuntu13.5
→ Debian/Ubuntu Packaging Revision
```

---

## Epoch

### 한 줄 정의

**기존 Version 체계만으로 순서를 올바르게 비교하기 어려울 때 우선순위를 보정하기 위한 값이다.**

예:

```text
2:1.0
```

의 `2:`가 Epoch다.

초보 단계에서는:

```text
Epoch
→ Version 비교에서 우선적으로 고려되는 보정값
```

정도로 이해하면 된다.

---

## Upstream Version

원래 소프트웨어 프로젝트에서 사용하는 Version이다.

예:

```text
9.6p1
```

---

## Debian/Ubuntu Revision

Debian/Ubuntu가 해당 소프트웨어를 Package로 만들면서 적용한 수정/배포 Revision을 나타내는 부분이다.

중요:

> **APT/dpkg는 단순 문자열 비교가 아니라 Debian Version 비교 규칙에 따라 Version 전체를 비교한다.**

---

# 5. `apt upgrade`

```bash
sudo apt upgrade
```

## 한 줄 정의

**설치된 Package를 가능한 새 Version으로 업그레이드하되, 기존 Package 제거가 필요한 변경은 수행하지 않는 방향으로 처리한다.**

중요한 특징:

```text
기존 Package 업그레이드
→ 가능

필요한 새 Dependency 추가
→ 상황에 따라 가능

기존 설치 Package 제거
→ 하지 않는 방향
```

그래서 Dependency 관계상 기존 Package 제거가 필요하면 일부 Package가 보류될 수 있다.

출력에서 다음과 같은 메시지를 볼 수 있다.

```text
kept back
```

또는 현재 APT의 표현 방식에 따라 별도 보류/단계적 업데이트 안내가 나타날 수 있다.

---

# 6. `apt full-upgrade`

```bash
sudo apt full-upgrade
```

## 한 줄 정의

**전체 Dependency 관계를 해결하기 위해 필요하면 새 Package 설치뿐 아니라 기존 Package 제거까지 허용하는 업그레이드 방식이다.**

따라서:

```text
새 Dependency 설치
기존 Dependency 교체
기존 Package 제거
```

가 발생할 수 있다.

즉 `upgrade`보다 변경 범위가 커질 수 있다.

---

# 7. `upgrade` vs `full-upgrade`

| 항목 | `apt upgrade` | `apt full-upgrade` |
|---|---|---|
| 설치된 Package 업그레이드 | O | O |
| 새 Dependency 설치 | 가능 | 가능 |
| 기존 Package 제거 | 하지 않는 방향 | 필요하면 가능 |
| 변경 범위 | 비교적 보수적 | 더 넓을 수 있음 |
| 운영 전 검토 | 필요 | 특히 중요 |

핵심:

```text
upgrade
→ 기존 설치 구성을 최대한 유지

full-upgrade
→ 전체 Dependency 해결을 위해 제거까지 허용
```

> `full-upgrade`가 무조건 더 좋거나 더 최신이라는 뜻은 아니다. 변경 정책이 더 적극적이라는 의미다.

---

# 8. `--dry-run`

실제 변경 전에 예정 작업을 확인할 수 있다.

```bash
sudo apt upgrade --dry-run
```

```bash
sudo apt full-upgrade --dry-run
```

## 한 줄 정의

**실제 Package 변경은 하지 않고, 수행 예정 작업을 계산해 보여준다.**

확인할 내용:

```text
몇 개가 업그레이드되는가?
새로 설치되는 Package가 있는가?
제거되는 Package가 있는가?
보류되는 Package가 있는가?
중요 Service / Kernel Package가 포함되는가?
```

특히 `full-upgrade`에서는:

```text
REMOVED
```

대상이 있는지 반드시 확인한다.

---

# 9. 특정 Package만 업그레이드

예:

```bash
sudo apt install --only-upgrade openssh-client
```

## 의미

```text
apt install
→ Package 설치/Version 선택 기능 사용

--only-upgrade
→ 이미 설치된 Package만 업그레이드
→ 미설치 Package를 새로 설치하지 않음
```

특정 Package만 제한적으로 업데이트할 때 유용하다.

업데이트 전:

```bash
apt policy openssh-client
```

로 Installed / Candidate를 확인하는 습관이 좋다.

---

# 10. Package hold

## 한 줄 정의

**hold는 특정 Package를 일반적인 자동 업그레이드 대상에서 고정해 현재 Version을 유지하도록 하는 상태다.**

설정:

```bash
sudo apt-mark hold tree
```

확인:

```bash
apt-mark showhold
```

해제:

```bash
sudo apt-mark unhold tree
```

---

## 왜 hold가 필요한가?

운영 환경에서는 최신 Version이 항상 즉시 적용 가능한 것은 아니다.

예:

```text
Application 호환성 검증 전
특정 Runtime Version에 의존
Vendor가 특정 Version만 지원
신규 Version 장애 사례 확인 중
변경 승인 전
```

이럴 때:

```text
현재 Version 유지
↓
검증
↓
변경 승인
↓
unhold
↓
업데이트
```

흐름을 사용할 수 있다.

---

## hold 흐름

```text
apt-mark hold PACKAGE
↓
Package Version 고정

apt-mark showhold
↓
현재 hold Package 확인

apt-mark unhold PACKAGE
↓
hold 해제
↓
다시 일반 업그레이드 가능
```

> hold는 운영 통제 수단이지 문제를 해결하는 명령은 아니다. 왜 해당 Version을 고정했는지 기록하는 것이 중요하다.

---

# 11. 특정 Version 지정 설치

형식:

```bash
sudo apt install PACKAGE=VERSION
```

예시 형식:

```bash
sudo apt install nginx=<원하는-version>
```

단, 해당 Version이 APT가 접근 가능한 Repository/Package Source에 존재해야 한다.

먼저:

```bash
apt policy nginx
```

로 사용 가능한 Version을 확인한다.

개념:

```text
apt policy
→ 가능한 Version 확인

apt install PACKAGE=VERSION
→ 원하는 Version 명시
```

주의:

> 특정 Version 강제 변경은 Dependency 호환성에 영향을 줄 수 있으므로 운영 서버에서는 영향 검토가 필요하다.

---

# 12. `apt`와 `apt-get`

둘 다 APT 계열 Package 관리 도구다.

```text
apt
→ 사람이 직접 사용하는 대화형 CLI에 편리

apt-get
→ 전통적인 APT CLI
→ Script / 자동화 환경에서 많이 사용
```

초기 학습은 `apt` 중심으로 진행한다.

자동화에서는 출력 형식과 동작 안정성을 고려해 `apt-get`을 사용하는 경우가 많다.

---

# 13. Service Package 업데이트

예:

```text
nginx
openssh-server
mysql-server
```

같은 Service Package가 업데이트되면 단순 파일 교체로 끝나지 않을 수 있다.

가능한 영향:

```text
Binary 교체
Library 교체
설정 호환성 변화
Service restart
Connection 영향
```

따라서 업데이트 후에는 Day 6에서 배운 검증을 연결한다.

```bash
systemctl status SERVICE
journalctl -u SERVICE
```

그리고 필요하면:

```text
Process
Port
실제 Application 기능
```

까지 확인한다.

핵심:

```text
Package 업데이트 성공
≠
Service 정상 동작 보장
```

---

# 14. Kernel Package 업데이트

Kernel도 Package로 관리된다.

예:

```text
linux-image-...
linux-headers-...
```

현재 실행 중 Kernel 확인:

```bash
uname -r
```

Kernel Package가 새로 설치되더라도:

```text
새 Kernel Package 설치
≠
현재 실행 중 Kernel 즉시 교체
```

인 경우가 일반적이다.

새 Kernel로 부팅하려면 재부팅이 필요할 수 있다.

---

# 15. Reboot 필요 여부

Ubuntu에서는 업데이트 결과에 따라 다음 파일이 생성될 수 있다.

```text
/var/run/reboot-required
```

확인 예:

```bash
test -f /var/run/reboot-required && cat /var/run/reboot-required
```

명령 구조:

```text
test -f FILE
→ 일반 파일이 존재하는지 검사

&&
→ 앞 명령이 성공했을 때만 다음 명령 실행

cat
→ 파일 내용 출력
```

파일이 존재하면 재부팅이 필요한 변경이 있었음을 판단하는 하나의 신호로 사용할 수 있다.

---

# 16. Package 파일과 실행 중 Process의 관계

이 부분은 Day 5 Process와 연결된다.

예를 들어 실행 중인 Service의 Binary 파일이 Package 업데이트로 교체되었다고 하자.

```text
Disk의 Binary
→ 새 Version으로 교체

이미 실행 중인 Process
→ 기존에 로드한 Code/Memory 상태로 계속 실행 가능
```

즉:

> **Disk의 실행 파일이 새 Version으로 바뀌었다고 실행 중 Process가 자동으로 새 코드로 변하는 것은 아니다.**

그래서 Package에 따라 Service restart가 필요할 수 있다.

연결:

```text
Package 관리
→ Disk의 파일/Version 관리

Process
→ 현재 실행 중인 실행 Context

systemd
→ Service Lifecycle 관리
```

Day 5, Day 6, Day 7이 여기서 연결된다.

---

# 17. 업데이트 출력에서 확인할 문구

APT 출력에서는 다음 부분을 주의해서 본다.

```text
The following packages will be upgraded:
The following NEW packages will be installed:
The following packages will be REMOVED:
...
```

운영 관점 우선순위:

```text
REMOVED
→ 기존 Package 제거 여부 확인

NEW
→ 새로운 Dependency/Package 추가 여부

upgraded
→ 실제 변경 Package 목록

보류 대상
→ 왜 적용되지 않는지 확인
```

출력 문구는 APT Version에 따라 조금 달라질 수 있다.

---

# 18. 💼 운영 서버 업데이트 절차

권장 흐름:

```text
1. sudo apt update
2. apt list --upgradable
3. 중요 Package의 apt policy 확인
4. apt-mark showhold
5. upgrade/full-upgrade --dry-run
6. NEW / REMOVED / Service / Kernel 영향 확인
7. 변경 승인 및 작업 시간 확인
8. 실제 upgrade 수행
9. Package Version 재확인
10. Service 상태 확인
11. journal 확인
12. Process / Port 확인
13. 실제 Application 기능 검증
14. reboot 필요 여부 확인
15. 변경 내역 기록
```

중요:

> **업데이트 성공 여부는 APT 명령의 Exit Status만이 아니라 서비스와 실제 기능까지 확인해야 한다.**

---

# 19. 🔧 Troubleshooting 시작점

Package가 예상대로 업데이트되지 않는다면 다음 가설을 세운다.

```text
Candidate가 존재하지 않는가?
Local Package Index가 오래됐는가?
hold 상태인가?
Dependency 충돌이 있는가?
Package가 보류됐는가?
Repository 우선순위 문제인가?
broken Package 상태인가?
```

기본 확인:

```bash
apt policy PACKAGE
apt-mark showhold
apt list --upgradable
sudo apt update
sudo apt upgrade --dry-run
```

Day 7-5에서 broken package, dpkg lock, interrupted configuration 등을 더 깊게 다룬다.

---

# 20. 🧪 Day 7-4 실습

## 1) 업데이트 가능 Package 확인

```bash
apt list --upgradable
```

## 2) hold Package 확인

```bash
apt-mark showhold
```

## 3) 특정 Package의 Version 상태 확인

```bash
apt policy openssh-client
```

## 4) upgrade 미리보기

```bash
sudo apt upgrade --dry-run
```

## 5) full-upgrade 미리보기

```bash
sudo apt full-upgrade --dry-run
```

두 결과에서:

```text
UPGRADED
NEW
REMOVED
보류
```

차이를 비교한다.

---

# 21. hold / unhold 실습

실습용 Package인 `tree`가 설치되어 있다면:

```bash
sudo apt-mark hold tree
```

확인:

```bash
apt-mark showhold
```

해제:

```bash
sudo apt-mark unhold tree
```

다시 확인:

```bash
apt-mark showhold
```

확인 목표:

```text
hold 적용
→ showhold에서 확인

unhold 적용
→ showhold에서 사라짐
```

---

# 22. 명령어 빠른 복습

| 명령 | 핵심 목적 |
|---|---|
| `apt list --upgradable` | 업데이트 가능한 Package 확인 |
| `apt policy PACKAGE` | Installed / Candidate / Version Source 확인 |
| `sudo apt upgrade` | 기존 Package 제거 없이 가능한 범위에서 업그레이드 |
| `sudo apt full-upgrade` | 필요하면 Package 제거까지 허용하며 Dependency 해결 |
| `--dry-run` | 실제 변경 전 예정 작업 확인 |
| `apt-mark showhold` | hold된 Package 확인 |
| `apt-mark hold PACKAGE` | Package Version 고정 |
| `apt-mark unhold PACKAGE` | hold 해제 |
| `apt install --only-upgrade PACKAGE` | 설치된 특정 Package만 업그레이드 |
| `apt install PACKAGE=VERSION` | 특정 Version 지정 |
| `uname -r` | 현재 실행 중 Kernel Version 확인 |

---

# 23. ⚠️ 헷갈리기 쉬운 부분

## `apt update` vs `apt upgrade`

```text
apt update
→ Package Metadata / Index 최신화

apt upgrade
→ 실제 설치된 Package Version 변경
```

---

## Candidate가 높다고 즉시 업데이트해야 하는가?

아니다.

운영 환경에서는:

```text
호환성
Service 영향
Kernel 여부
작업 시간
Rollback 계획
Vendor 지원
```

까지 고려한다.

---

## `full-upgrade`가 더 좋은 명령인가?

아니다.

```text
upgrade
→ 더 보수적인 변경 정책

full-upgrade
→ Dependency 해결을 위해 제거까지 허용
```

목적과 영향 범위가 다르다.

---

## hold는 삭제인가?

아니다.

```text
hold
→ Package는 설치된 채 Version 변경을 제한

remove
→ Package 자체 제거
```

완전히 다른 개념이다.

---

## Package 업데이트 후 Process도 자동으로 새 Version인가?

항상 그렇지 않다.

```text
Disk의 Binary 교체
≠
이미 실행 중 Process의 Memory Image 자동 교체
```

Service 특성에 따라 restart/reload 또는 reboot가 필요할 수 있다.

---

# 24. ✅ 핵심 정리

```text
apt update
→ 최신 Package Metadata 확보

apt list --upgradable
→ Installed < Candidate 대상 확인

apt policy PACKAGE
→ 현재/후보 Version 확인

apt upgrade
→ 기존 Package 제거 없이 가능한 업그레이드

apt full-upgrade
→ 필요하면 Package 제거까지 허용

--dry-run
→ 실제 변경 전에 영향 확인

apt-mark hold
→ Version 고정

apt-mark unhold
→ 고정 해제
```

운영 흐름:

```text
Update Metadata
→ Upgrade 대상 확인
→ Version / hold 확인
→ Dry Run
→ 영향 분석
→ 변경
→ Package Version 검증
→ Service / Process / Log 검증
→ 실제 기능 검증
→ Reboot 필요 여부 확인
```

가장 중요한 문장:

> **운영 서버의 Package 업데이트는 "최신으로 만드는 작업"이 아니라 "변경 범위를 파악하고 통제하면서 안전하게 Version을 전환하는 작업"이다.**

---

# 25. 🧠 복습 문제

1. `apt list --upgradable`은 어떤 Package를 보여주는가?
2. Installed와 Candidate의 차이는 무엇인가?
3. Debian/Ubuntu Package Version의 Epoch는 왜 필요한가?
4. `apt update`와 `apt upgrade`의 차이는?
5. `apt upgrade`는 기존 Package 제거가 필요한 경우 어떻게 처리하는가?
6. `apt full-upgrade`가 `upgrade`보다 영향 범위가 클 수 있는 이유는?
7. `--dry-run`을 운영 환경에서 사용하는 이유는?
8. 특정 설치 Package 하나만 업그레이드할 때 어떤 옵션을 사용할 수 있는가?
9. hold는 무엇이며 어떤 상황에서 사용하는가?
10. hold된 Package 목록은 어떻게 확인하는가?
11. hold를 해제하는 명령은 무엇인가?
12. 특정 Version을 지정하는 기본 문법은?
13. Service Package 업데이트 후 `systemctl status`와 `journalctl`을 확인하는 이유는?
14. Kernel Package가 업데이트된 직후 현재 실행 Kernel이 자동으로 바뀌지 않을 수 있는 이유는?
15. Package Binary가 바뀌어도 실행 중 Process가 즉시 새 코드로 바뀌지 않는 이유는?
16. 운영 서버에서 실제 Upgrade 전에 확인해야 할 최소 항목은 무엇인가?

---

## 다음 학습

**Day 7-5 — Package 장애 복구와 Troubleshooting**

다음에는 다음을 다룬다.

```text
Broken Dependency
dpkg Configuration 중단
apt/dpkg Lock
dpkg --configure -a
apt --fix-broken install
Package install 실패
Repository / Dependency / Lock 문제 구분
안전한 복구 순서
```
