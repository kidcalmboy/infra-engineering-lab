# Day 7-5 — Package 장애 복구와 Troubleshooting

> APT/dpkg 오류를 무작정 명령으로 덮는 것이 아니라, **오류 문구를 기준으로 Repository / Dependency / dpkg configure / Lock / Candidate 문제로 분류하고 최소 조치 후 재검증하는 운영형 복구 흐름**을 정리한다.

## 📌 이번에 배운 내용

- Package 장애를 유형별로 분류하는 방법
- apt와 dpkg 오류의 관계
- Broken Dependency
- `apt --fix-broken install`
- dpkg configure 중단 상태
- `dpkg --configure -a`
- apt/dpkg Lock의 의미
- Lock 파일을 함부로 삭제하면 안 되는 이유
- `pgrep -af 'apt|dpkg'`
- Repository 오류와 Dependency 오류 구분
- `has no installation candidate`
- `dpkg --audit`
- `apt clean`, `apt autoclean`
- 오류 문구별 1차 진단
- 안전한 복구 순서
- 복구 후 실제 기능 검증

## 📚 목차

1. Package 장애 전체 분류
2. apt와 dpkg의 역할 연결
3. Broken Dependency
4. `apt --fix-broken install`
5. configure 단계와 중단 상태
6. `dpkg --configure -a`
7. 두 복구 명령의 차이
8. apt/dpkg Lock
9. Lock 장애 접근 순서
10. Repository 문제
11. Candidate 없음
12. `dpkg --audit`
13. Package Cache 정리
14. 오류 문구별 1차 분류
15. Troubleshooting Level 1~4
16. 안전한 관찰 실습
17. 핵심 정리
18. 복습 문제

---

# 1. Package 장애 전체 분류

Package 장애는 먼저 유형을 나눠야 한다.

```text
Repository 문제
Dependency 문제
dpkg configure 중단
apt/dpkg Lock
Package Database 불완전 상태
Package 이름 / Version / Candidate 문제
```

잘못된 접근:

```text
apt 오류
→ 무조건 --fix-broken
→ 안 되면 lock 파일 삭제
→ 안 되면 reboot
```

권장 접근:

```text
증상
→ Error Message 확인
→ 문제 유형 분류
→ 상태 확인
→ 원인 가설
→ 최소 조치
→ 재검증
→ 실제 기능 검증
```

---

# 2. apt와 dpkg의 역할 연결

Day 7-1에서 배운 구조:

```text
apt
→ Repository / Dependency / Download / 고수준 관리

dpkg
→ 실제 .deb unpack / configure / 로컬 Package 상태 관리
```

예:

```text
apt install
↓
.deb 다운로드
↓
dpkg가 Package 처리
↓
중간에 작업 중단
```

이 경우 Repository는 정상이어도 **dpkg 상태가 불완전**할 수 있다.

즉:

> apt 명령에서 오류가 보여도 실제 원인이 dpkg 단계일 수 있다.

---

# 3. Broken Dependency

## 한 줄 정의

**현재 설치 상태가 어떤 Package가 요구하는 Dependency 조건을 만족하지 못하는 상태**다.

예:

```text
Package A
→ libB >= 2.0 필요

현재 시스템
→ libB 1.5
```

대표 메시지:

```text
Unmet dependencies
Depends: ...
but it is not going to be installed
```

확인 포인트:

```text
어떤 Dependency가 필요한가?
현재 Installed Version은?
Candidate가 존재하는가?
Repository에서 해당 Version을 제공하는가?
```

---

# 4. `apt --fix-broken install`

```bash
sudo apt --fix-broken install
```

## 한 줄 정의

**깨진 Dependency 관계를 해결하기 위해 apt가 필요한 설치/제거 작업을 계산하고 복구를 시도하는 명령**이다.

개념 흐름:

```text
현재 apt/dpkg 상태 확인
↓
깨진 Dependency 파악
↓
필요한 Package 설치/조정 계산
↓
Dependency 정상화 시도
```

중요:

> `--fix-broken`은 무조건 안전한 복구 버튼이 아니다.

Package 추가 설치나 제거가 발생할 수 있으므로 가능하면 먼저:

```bash
sudo apt --fix-broken install --dry-run
```

으로 영향 범위를 확인한다.

---

# 5. Package configure 단계

Package 설치는 단순 파일 복사만이 아니다.

```text
download
↓
unpack
↓
configure
```

## configure란?

**Unpack된 Package를 실제 사용 가능한 상태로 마무리 설정하는 단계**다.

여기에는 Package에 따라 다음 작업이 포함될 수 있다.

```text
설정 파일 처리
Maintainer Script 실행
Service 관련 후처리
Dependency 기반 설정
기타 설치 마무리 작업
```

따라서:

```text
파일이 일부 존재
≠
Package configure 완료
```

일 수 있다.

---

# 6. `dpkg --configure -a`

```bash
sudo dpkg --configure -a
```

명령 구조:

```text
dpkg
→ 저수준 Debian Package 관리 도구

--configure
→ configure 작업 수행

-a
→ all
```

## 한 줄 정의

**Unpack은 되었지만 configure가 끝나지 않은 모든 Package의 설정 작업을 마무리하도록 dpkg에 요청한다.**

대표 안내:

```text
dpkg was interrupted
```

---

# 7. `dpkg --configure -a` vs `apt --fix-broken install`

| 명령 | 주된 목적 |
|---|---|
| `dpkg --configure -a` | configure가 끝나지 않은 Package 설정 마무리 |
| `apt --fix-broken install` | 깨진 Dependency 관계 해결 |

자주 볼 수 있는 흐름:

```text
dpkg was interrupted
↓
dpkg --configure -a
↓
Dependency 문제 남음?
↓
apt --fix-broken install
↓
상태 재검증
```

단, 항상 기계적으로 이 순서를 쓰지 말고 **실제 오류 메시지에 맞춰 선택**한다.

---

# 8. apt/dpkg Lock

## 한 줄 정의

**여러 Package 관리 Process가 동시에 Package Database를 수정하지 못하도록 막는 잠금 장치**다.

Lock이 필요한 이유:

```text
apt install
dpkg configure
apt upgrade
```

같은 작업이 동시에 같은 Package Database를 수정하면 상태가 손상될 수 있기 때문이다.

즉:

```text
Lock
→ 문제의 원인 그 자체라기보다 Package Database 보호 장치
```

대표 오류:

```text
Could not get lock
Unable to acquire the dpkg frontend lock
```

---

# 9. Lock 장애 접근 순서

가장 먼저 다른 Package Manager Process가 실제로 동작 중인지 확인한다.

```bash
pgrep -af 'apt|dpkg'
```

또는:

```bash
ps aux | grep -E 'apt|dpkg'
```

확인할 것:

```text
apt/dpkg가 실제 실행 중인가?
정상 설치/업데이트가 진행 중인가?
백그라운드 자동 업데이트가 동작 중인가?
```

Ubuntu에서는 `apt-daily`, `unattended-upgrades` 같은 작업이 Package 관리와 관련될 수 있다.

정상 Process가 Lock을 잡고 있다면:

```text
기다림
→ Process 종료 확인
→ 다시 시도
```

가 맞는 경우가 많다.

---

# 10. Lock 파일을 바로 삭제하면 안 되는 이유

인터넷 검색에서 다음과 비슷한 명령을 볼 수 있다.

```bash
sudo rm /var/lib/dpkg/lock-frontend
```

하지만 **원인 확인 없이 바로 실행하면 안 된다.**

실제 Package Manager Process가 살아 있는데 Lock만 삭제하면:

```text
Process A가 Package Database 수정 중
↓
Lock 강제 삭제
↓
Process B도 수정 시작
↓
동시 수정
↓
Package Database 손상 가능
```

핵심:

> **Lock 파일은 보호 장치다. Lock이 보인다고 Lock 파일 자체가 문제라는 뜻은 아니다.**

---

# 11. Repository 문제

대표 메시지:

```text
Temporary failure resolving
404 Not Found
NO_PUBKEY
Signature verification failed
```

1차 분류:

```text
Temporary failure resolving
→ DNS / Network

404 Not Found
→ Repository URL / Suite / Release 지원 여부

NO_PUBKEY
→ Signing Key / Repository Trust

Signature verification failed
→ Metadata 검증 문제
```

Broken Dependency와는 다른 문제다.

---

# 12. `has no installation candidate`

대표 메시지:

```text
Package ... has no installation candidate
```

## 의미

**현재 APT Package Index 기준으로 설치 후보인 Candidate를 찾을 수 없다는 뜻**이다.

가능 원인:

```text
Package 이름 오류
apt update가 오래됨
Repository 비활성
해당 Ubuntu Release에서 Package 미제공
외부 Repository 누락
Architecture 조건
Repository 우선순위/구성 문제
```

확인:

```bash
apt policy PACKAGE
apt search PACKAGE
sudo apt update
```

필요하면 Repository 설정도 확인한다.

---

# 13. `dpkg --audit`

```bash
sudo dpkg --audit
```

## 한 줄 정의

**설치 상태가 불완전하거나 주의가 필요한 Package 상태를 점검한다.**

정상 상태라면 출력이 없을 수도 있다.

사용 목적:

```text
configure가 안 끝난 Package가 있는가?
Package 상태가 불완전한가?
dpkg Database 기준으로 이상 상태가 있는가?
```

복구 전과 복구 후 모두 유용하다.

---

# 14. Package Cache 정리

APT Package Archive Cache는 보통:

```text
/var/cache/apt/archives/
```

와 관련된다.

## `apt clean`

```bash
sudo apt clean
```

**다운로드된 Package Archive Cache를 정리한다.**

```text
설치된 Package 삭제 X
dpkg Database 삭제 X
다운로드 Cache 정리
```

## `apt autoclean`

```bash
sudo apt autoclean
```

초보 단계에서는:

```text
clean
→ Package Archive Cache를 넓게 정리

autoclean
→ 더 이상 내려받기 어려운 오래된 Cache 중심 정리
```

정도로 이해한다.

Cache 정리는 Dependency/Lock/Repository 문제의 만능 해결책이 아니다.

---

# 15. 오류 문구별 1차 분류

| 오류 문구 | 우선 의심 영역 |
|---|---|
| `Temporary failure resolving` | DNS / Network |
| `404 Not Found` | Repository URL / Suite / Release |
| `NO_PUBKEY` | Signing Key |
| `Signature verification failed` | Repository Metadata 검증 |
| `Unmet dependencies` | Dependency |
| `dpkg was interrupted` | configure 중단 |
| `Could not get lock` | 다른 apt/dpkg Process |
| `has no installation candidate` | Repository / Package availability |

이 표의 목적은 바로 해결 명령을 고르는 것이 아니라 **가설의 범위를 빠르게 좁히는 것**이다.

---

# 16. Troubleshooting Level 1 — 증상과 상태 확인

기본 확인:

```bash
apt policy PACKAGE
dpkg -l | grep PACKAGE
sudo dpkg --audit
pgrep -af 'apt|dpkg'
```

필요하면 Repository 상태:

```bash
sudo apt update
```

핵심:

```text
명령 실행
→ 결과 해석
→ 다음 가설 선택
```

---

# 17. Troubleshooting Level 2 — 원인별 최소 조치

## configure 중단

```bash
sudo dpkg --configure -a
```

## Broken Dependency

```bash
sudo apt --fix-broken install
```

가능하면 먼저:

```bash
sudo apt --fix-broken install --dry-run
```

## Lock

```text
Process 확인
→ 정상 작업이면 기다림
→ 비정상 상태인지 판단
→ 원인 확인 후 조치
```

## Repository

```text
apt update 출력 확인
→ Network / DNS
→ Repository URL
→ Suite
→ Signature/Key
→ 지원 여부
```

---

# 18. Troubleshooting Level 3 — 재검증

조치 후:

```bash
sudo dpkg --audit
apt policy PACKAGE
dpkg -l | grep PACKAGE
```

필요하면 설치 재시도:

```bash
sudo apt install PACKAGE
```

복구 명령이 성공했다는 것만으로 Package 기능까지 정상이라고 판단하지 않는다.

---

# 19. Troubleshooting Level 4 — 실제 기능 검증

CLI Package:

```bash
command -v PROGRAM
PROGRAM --version
```

Service Package:

```bash
systemctl status SERVICE
journalctl -u SERVICE
```

필요하면:

```text
Process
Port
Config
Application 기능
```

까지 확인한다.

---

# 20. 전체 복구 흐름

```text
증상
↓
Error Message 확인
↓
유형 분류

Repository?
Dependency?
dpkg configure?
Lock?
Candidate 없음?

↓
상태 확인

apt policy
dpkg -l
dpkg --audit
pgrep apt/dpkg

↓
원인 확정
↓
최소 조치

dpkg --configure -a
apt --fix-broken install
Repository 수정
정상 Package Manager Process 종료 대기

↓
재검증

dpkg --audit
apt policy
dpkg -l

↓
실제 기능 검증

CLI 실행
또는
systemctl / journal / Port / Application
```

---

# 21. 재부팅부터 하지 않는 이유

다음 문제는 재부팅만으로 해결되지 않을 수 있다.

```text
Broken Dependency
dpkg configure 중단
Repository 오류
Signing Key 문제
Package Candidate 없음
```

따라서 Package 장애에서는 현재 Error Message와 Package 상태를 먼저 확인한다.

---

# 22. 이번 실습은 일부러 시스템을 깨지 않는다

학습을 위해 다음 행동은 하지 않는다.

```text
dpkg 강제 Kill
apt 설치 중 강제 종료
Lock 파일 강제 삭제
Dependency 의도적 손상
Package Database 직접 수정
```

목표는 장애를 만드는 것이 아니라 실제 장애 발생 시 올바른 순서로 대응하는 능력을 만드는 것이다.

---

# 23. 안전한 관찰 실습

```bash
sudo dpkg --audit
pgrep -af 'apt|dpkg'
apt policy tree
dpkg -l | grep tree
sudo apt --fix-broken install --dry-run
du -sh /var/cache/apt/archives/
```

확인 목표:

```text
불완전 Package 존재 여부
Package Manager Process 존재 여부
Installed / Candidate 상태
dpkg 설치 상태
Broken Dependency 복구 예정 작업
APT Cache 크기
```

실제 터미널 출력 원문이 공유되지 않았으므로 특정 결과를 실측값처럼 기록하지 않는다.

---

# 24. 명령어 빠른 복습

| 명령 | 목적 |
|---|---|
| `sudo dpkg --audit` | 불완전 Package 상태 점검 |
| `sudo dpkg --configure -a` | 미완료 configure 작업 마무리 |
| `sudo apt --fix-broken install` | Broken Dependency 해결 시도 |
| `sudo apt --fix-broken install --dry-run` | 복구 영향 미리보기 |
| `pgrep -af 'apt\|dpkg'` | apt/dpkg Process 확인 |
| `apt policy PACKAGE` | Installed / Candidate / Version 확인 |
| `dpkg -l \| grep PACKAGE` | 로컬 dpkg 설치 상태 확인 |
| `sudo apt clean` | Package Archive Cache 정리 |
| `sudo apt autoclean` | 오래되거나 불필요해진 Cache 정리 |

---

# 25. ⚠️ 절대 습관화하면 안 되는 패턴

```text
apt 오류
→ 무조건 Lock 파일 삭제

Dependency 오류
→ 무조건 --fix-broken

Package install 실패
→ 무조건 reboot

Repository 오류
→ 아무 Key나 추가

Package 상태 이상
→ 원인 확인 없이 purge/reinstall 반복
```

대신:

```text
Error Message
→ 분류
→ 상태 확인
→ 원인
→ 최소 조치
→ 재검증
```

을 기본 패턴으로 사용한다.

---

# 26. ✅ Day 7 전체 연결

```text
Day 7-1
Package / Repository / apt / dpkg 기초

Day 7-2
Repository 구조 / apt update / Metadata

Day 7-3
install / remove / purge / dpkg 상태

Day 7-4
upgrade / Version / hold / Kernel 영향

Day 7-5
Broken Dependency / configure / Lock / 장애 복구
```

Day 7 전체 운영 흐름:

```text
Repository 이해
→ Package 정보 갱신
→ 설치
→ 상태/파일 확인
→ 삭제
→ 업데이트
→ Version 통제
→ 장애 원인 분류
→ 복구
→ 실제 기능 검증
```

---

# 27. 🧠 복습 문제

1. apt 오류처럼 보여도 실제 원인이 dpkg 단계일 수 있는 이유는?
2. Broken Dependency란 무엇인가?
3. `apt --fix-broken install`은 어떤 문제를 해결하려는 명령인가?
4. `dpkg --configure -a`는 어떤 상태에서 사용하는가?
5. 두 복구 명령의 역할 차이는?
6. apt/dpkg Lock은 왜 필요한가?
7. Lock 파일을 바로 삭제하면 위험한 이유는?
8. `pgrep -af 'apt|dpkg'`로 무엇을 확인하는가?
9. `Temporary failure resolving`과 `Unmet dependencies`는 각각 어떤 영역을 먼저 의심해야 하는가?
10. `has no installation candidate`의 의미는?
11. `dpkg --audit`의 목적은?
12. `apt clean`이 설치된 Package를 삭제하는 명령이 아닌 이유는?
13. Package 문제에서 재부팅부터 하지 않는 이유는?
14. 복구 후 어떤 명령으로 상태를 재검증할 수 있는가?
15. Service Package라면 복구 후 Day 6의 어떤 검증을 연결해야 하는가?
16. Package Troubleshooting의 기본 순서를 말해보라.

---

## Day 7 완료

**Package 관리 파트 완료.**

다음 학습은 **Day 8 — Disk / Filesystem**이다.

```text
Block Device
Partition
Filesystem
Mount
lsblk
df
du
inode
/etc/fstab
LVM 기초
Disk Full 장애 분석
```

을 개념부터 실습까지 연결한다.
