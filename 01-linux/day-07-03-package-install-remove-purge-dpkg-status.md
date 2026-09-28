# Day 7-3 — 실제 Package 설치·삭제와 dpkg 상태 확인

> 작은 CLI 패키지인 `tree`를 예제로 사용해 **설치 전 상태 확인 → 설치 → 파일/버전 검증 → remove → purge → autoremove 사전 확인**까지 패키지 lifecycle을 정리한다.

## 📌 이번에 배운 내용

- 설치 전 `apt policy`로 상태와 Candidate 확인
- `apt install`이 실제로 수행하는 흐름
- `dpkg -l`의 상태 코드와 `ii`, `rc`
- `dpkg -L`: Package → Files
- `dpkg -S`: File → Package
- `command -v` / `which`로 실행 파일 위치 확인
- `apt remove`와 `apt purge` 차이
- `apt autoremove`의 목적
- `--dry-run`으로 변경 전 영향 확인
- apt 관점과 dpkg 관점의 차이
- 패키지 설치 후 실제 기능 검증이 필요한 이유
- 운영 서버에서 안전하게 설치·삭제하는 흐름

## 📚 목차

1. 실습 패키지와 목표
2. 설치 전 상태 확인
3. `apt install` 내부 흐름
4. `dpkg -l` 상태 코드
5. Package가 설치한 파일 확인
6. File이 어느 Package 소속인지 확인
7. 실행 파일과 실제 기능 검증
8. `apt remove`
9. `apt purge`
10. `rc` 상태
11. `apt autoremove`
12. `--dry-run`
13. apt와 dpkg 관점 비교
14. 운영형 설치/삭제 절차
15. 헷갈리기 쉬운 부분
16. 핵심 정리
17. 복습 문제

---

# 1. 실습 패키지와 목표

이번 파트에서는 작은 CLI 패키지인:

```text
tree
```

를 예제로 사용한다.

`tree`를 실습 대상으로 선택하기 좋은 이유:

```text
용량이 작음
Service가 아님
systemd 영향이 없음
설치/삭제 결과를 확인하기 쉬움
실제 실행 검증이 간단함
```

이번 파트의 핵심 흐름:

```text
설치 전 확인
→ 설치
→ dpkg 상태 확인
→ 설치 파일 확인
→ 실행 파일 위치 확인
→ 실제 동작 확인
→ remove
→ purge
→ autoremove 사전 확인
```

---

# 2. 설치 전 상태 확인

먼저 패키지가 현재 설치되어 있는지 확인한다.

```bash
dpkg -l | grep '^ii' | grep tree
```

또는:

```bash
apt list --installed 2>/dev/null | grep '^tree/'
```

설치 후보 버전 확인:

```bash
apt policy tree
```

여기서 대표적으로 확인하는 값:

```text
Installed
→ 현재 설치된 버전

Candidate
→ 현재 APT가 설치 대상으로 선택할 버전

Version table
→ Repository에서 사용 가능한 버전 정보
```

미설치 상태라면 환경에 따라:

```text
Installed: (none)
```

처럼 보일 수 있다.

---

# 3. `apt install` 내부 흐름

설치:

```bash
sudo apt install tree
```

개념적 내부 흐름:

```text
Local Package Index 확인
↓
tree Package 정보 조회
↓
Candidate Version 결정
↓
Dependency 계산
↓
Repository에서 필요한 .deb 다운로드
↓
dpkg 계층으로 설치
↓
파일 배치
↓
dpkg Database에 설치 상태 기록
```

즉 `apt install`은 단순히 파일 하나를 복사하는 명령이 아니다.

---

# 4. `dpkg -l` 상태 코드

설치 후:

```bash
dpkg -l | grep tree
```

를 사용해 dpkg Database의 설치 상태를 확인할 수 있다.

정상 설치 상태에서는 흔히:

```text
ii
```

가 보인다.

## `ii`의 의미

`dpkg -l` 앞의 두 글자는 각각 상태 정보를 나타낸다.

초보 단계에서는:

```text
첫 번째 문자
→ Desired 상태

두 번째 문자
→ 현재 Package 상태
```

로 이해하면 된다.

```text
ii
││
│└─ installed
└── install을 원하는 상태
```

즉 정상적으로 설치된 Package라고 이해하면 된다.

> dpkg 상태 코드는 더 다양하지만 Day 7-3에서는 `ii`, `rc`를 우선 이해한다.

---

# 5. Package가 설치한 파일 확인 — `dpkg -L`

```bash
dpkg -L tree
```

## 한 줄 정의

**특정 Package가 시스템에 설치한 파일 목록을 확인한다.**

즉 방향은:

```text
Package
↓
Files
```

이다.

예를 들어 출력에:

```text
/usr/bin/tree
```

같은 실행 파일 경로가 포함될 수 있다.

실무에서는:

```text
이 Package가 어떤 Binary를 설치했는가?
설정 파일은 어디에 있는가?
문서 파일은 어디에 있는가?
```

를 확인할 때 유용하다.

---

# 6. File이 어느 Package 소속인지 확인 — `dpkg -S`

```bash
dpkg -S /usr/bin/tree
```

## 한 줄 정의

**특정 파일 경로가 어느 설치된 Debian Package에 속하는지 확인한다.**

방향:

```text
File
↓
Package
```

예상 형태:

```text
tree: /usr/bin/tree
```

따라서:

```text
dpkg -L
→ Package → Files

dpkg -S
→ File → Package
```

이 관계를 반드시 구분한다.

---

# 7. 실행 파일 위치와 실제 기능 검증

실행 파일 위치:

```bash
command -v tree
```

또는:

```bash
which tree
```

예상 경로:

```text
/usr/bin/tree
```

버전 확인:

```bash
tree --version
```

실제 기능 확인:

```bash
tree /etc | head -20
```

이 단계가 중요한 이유:

```text
Package 설치 완료
≠
실제 명령이 정상 실행된다는 것까지 자동 보장
```

따라서 설치 후에는:

```text
설치 상태
→ 실행 파일 위치
→ Version
→ 실제 실행
```

을 검증하는 습관이 좋다.

---

# 8. `apt remove`

```bash
sudo apt remove tree
```

## 한 줄 정의

**Package의 프로그램/실행 파일을 제거하는 데 중심을 둔 명령이다.**

중요:

```text
remove
≠
모든 Package 관련 파일 완전 삭제
```

Package에 따라 일부 설정 파일이 남을 수 있다.

제거 후 확인:

```bash
dpkg -l | grep tree
```

---

# 9. `apt purge`

```bash
sudo apt purge tree
```

## 한 줄 정의

**Package와 Package가 관리하는 설정 파일까지 제거하는 데 사용하는 명령이다.**

비교:

| 명령 | 의미 |
|---|---|
| `apt remove PACKAGE` | 실행 파일 등 Package 제거 중심 |
| `apt purge PACKAGE` | Package + Package가 관리하는 설정 파일 제거 |

주의:

> `purge`가 해당 프로그램과 관련된 사용자의 모든 데이터, 홈 디렉터리 파일, 애플리케이션 데이터까지 전부 지운다는 뜻은 아니다.

---

# 10. `rc` 상태

`dpkg -l`에서 경우에 따라:

```text
rc
```

를 볼 수 있다.

초보 단계에서 의미:

```text
r
→ Package 본체는 removed

c
→ config-files가 남아 있음
```

즉:

> **프로그램 파일은 제거됐지만 dpkg가 관리하는 설정 파일 일부가 남아 있는 상태**

라고 보면 된다.

이럴 때 `purge`를 사용해 남은 설정 파일을 제거할 수 있다.

---

# 11. `apt autoremove`

```bash
sudo apt autoremove
```

## 한 줄 정의

**다른 Package의 Dependency로 자동 설치되었지만 이제 더 이상 필요하지 않은 Package를 제거한다.**

예:

```text
Package A 설치
↓
Dependency B 자동 설치

나중에 A 제거
↓
B를 필요로 하는 다른 Package 없음
↓
B가 autoremove 대상이 될 수 있음
```

중요:

> 운영 서버에서는 바로 실행하지 말고 제거 예정 목록을 먼저 확인한다.

---

# 12. `--dry-run`

예:

```bash
sudo apt autoremove --dry-run
```

## 한 줄 정의

**실제 변경 없이 실행했을 때 어떤 작업이 수행될지 미리 보여준다.**

즉:

```text
실제 변경 X
↓
예상 영향 확인
↓
안전하다고 판단하면 실제 실행
```

운영 환경에서 매우 중요한 습관이다.

---

# 13. apt와 dpkg의 관점 차이

`apt policy tree`:

```text
Installed
Candidate
Repository Version
```

처럼 Repository와 설치 후보까지 포함한 **APT 관점**을 보여준다.

반면:

```bash
dpkg -l | grep tree
```

는 현재 시스템의 **dpkg Database 관점에서 실제 설치 상태**를 보여준다.

비교:

```text
apt
→ Repository + Candidate + Dependency + 설치 관리

dpkg
→ 현재 Debian Package 설치 상태와 파일 관리
```

---

# 14. 설치 lifecycle 전체

```text
apt policy tree
↓
현재 Installed / Candidate 확인
↓
apt install tree
↓
Local Index 확인
↓
Dependency 계산
↓
.deb 다운로드
↓
dpkg 설치
↓
파일 배치
↓
dpkg Database 기록
↓
dpkg -l / dpkg -L / dpkg -S 확인
↓
tree --version
↓
실제 기능 검증
```

---

# 15. 제거 lifecycle 전체

```text
apt remove tree
↓
Package 프로그램 제거
↓
설정 파일이 남을 수 있음
↓
dpkg -l에서 rc 가능

apt purge tree
↓
Package가 관리하는 남은 설정 파일 제거

apt autoremove --dry-run
↓
더 이상 필요하지 않은 자동 Dependency 후보 확인
```

---

# 16. 💼 운영형 설치/삭제 절차

## 설치

```text
1. Package 존재 여부 / 이름 확인
2. apt policy로 Installed / Candidate 확인
3. Dependency / 영향 범위 확인
4. apt install
5. dpkg 상태 확인
6. 실행 파일/설정 파일 위치 확인
7. Version 확인
8. 실제 기능 검증
9. Service Package라면 systemctl / journal까지 확인
```

## 삭제

```text
1. 어떤 Package를 제거하는지 확인
2. Dependency 영향 확인
3. remove / purge 차이 판단
4. 제거 예정 목록 확인
5. 제거
6. dpkg 상태 확인
7. 실행 파일/Service가 사라졌는지 검증
8. autoremove는 dry-run으로 영향 확인 후 결정
```

---

# 17. Service Package라면 Day 6와 연결

`tree`는 CLI 프로그램이므로 systemd Service가 없다.

하지만 nginx, sshd 같은 Service Package라면 설치 후:

```bash
systemctl status SERVICE
journalctl -u SERVICE
```

까지 확인해야 한다.

즉:

```text
Package 설치 성공
↓
Service Unit 존재 확인
↓
Service 상태 확인
↓
Log 확인
↓
Port / 실제 기능 확인
```

까지 이어져야 운영 검증이 끝난다.

---

# 18. ⚠️ 헷갈리기 쉬운 부분

## `apt remove`와 `apt purge`

```text
remove
→ Package 본체 제거 중심

purge
→ Package + Package 관리 설정 파일 제거
```

둘 다 사용자의 모든 데이터까지 자동으로 지운다는 의미는 아니다.

---

## `dpkg -L`과 `dpkg -S`

```text
dpkg -L PACKAGE
→ Package가 설치한 파일 찾기

dpkg -S PATH
→ 해당 파일을 설치한 Package 찾기
```

방향이 반대다.

---

## `apt policy`와 `dpkg -l`

```text
apt policy
→ Installed + Candidate + Repository 관점

dpkg -l
→ 로컬 dpkg Database 설치 상태 관점
```

---

## `autoremove`는 무조건 안전한가?

아니다.

APT가 더 이상 필요하지 않다고 판단한 자동 설치 Package가 제거 대상이 되지만 운영 환경에서는 실제 Application 영향까지 확인해야 한다.

그래서:

```bash
sudo apt autoremove --dry-run
```

으로 먼저 확인하는 습관이 중요하다.

---

# 19. 🔧 Troubleshooting 시작점

`apt install PACKAGE`가 실패하면 무작정 반복하지 않는다.

초기 확인 항목:

```text
1. Package 이름이 정확한가?
2. apt update가 정상적으로 끝났는가?
3. Repository에 Package가 존재하는가?
4. Candidate가 존재하는가?
5. Dependency 문제인가?
6. dpkg Database 상태가 정상인가?
7. 다른 apt/dpkg Process가 Lock을 잡고 있는가?
```

이 내용은 Day 7-5에서 실제 복구 명령과 함께 더 깊게 다룬다.

---

# 20. 🧪 Day 7-3 실습 명령

```bash
apt policy tree
sudo apt install tree

dpkg -l | grep tree
dpkg -L tree | head -20
command -v tree
dpkg -S /usr/bin/tree
tree --version

sudo apt remove tree
dpkg -l | grep tree

sudo apt purge tree
dpkg -l | grep tree

sudo apt autoremove --dry-run
```

### 실습에서 확인해야 하는 것

```text
설치 전 Installed / Candidate
설치 후 ii 상태
Package → File 관계
File → Package 관계
실행 파일 위치
실제 프로그램 실행 여부
remove 후 상태
purge 후 상태
autoremove가 제거하려는 대상
```

실제 터미널 출력 원문이 공유되지 않은 경우 특정 Version, Package 상태, 제거 목록을 실측값처럼 기록하지 않는다.

---

# 21. ✅ 핵심 정리

```text
apt policy PACKAGE
→ 설치 상태 + Candidate 확인

apt install PACKAGE
→ Package와 Dependency 실제 설치

dpkg -l
→ dpkg 설치 상태 확인

ii
→ 정상 설치 상태로 이해

rc
→ Package는 제거됐지만 config-files가 남은 상태

dpkg -L PACKAGE
→ Package → Files

dpkg -S PATH
→ File → Package

apt remove
→ Package 제거 중심

apt purge
→ Package + 관리 설정 파일 제거

apt autoremove
→ 더 이상 필요하지 않은 자동 Dependency 제거

--dry-run
→ 실제 변경 전 영향 미리보기
```

가장 중요한 운영 습관:

> **설치/삭제 명령이 성공했다는 것만 보지 말고, 전후 상태와 실제 기능을 반드시 검증한다.**

---

# 22. 🧠 복습 문제

1. `apt policy tree`에서 Installed와 Candidate는 각각 무엇인가?
2. `apt install`이 실제 설치하기 전 어떤 정보를 참고하는가?
3. `dpkg -l`의 `ii`는 초보 단계에서 어떻게 해석하면 되는가?
4. `rc`는 어떤 상태인가?
5. `dpkg -L tree`와 `dpkg -S /usr/bin/tree`의 방향 차이는?
6. `command -v tree`는 무엇을 확인하는가?
7. `apt remove`와 `apt purge`의 차이는?
8. `purge`가 사용자가 만든 모든 데이터를 자동 삭제한다는 의미인가?
9. `apt autoremove`는 어떤 Package를 제거하는가?
10. 운영 서버에서 `autoremove` 전에 `--dry-run`을 사용하는 이유는?
11. Package 설치 성공 후 실제 기능 검증이 필요한 이유는?
12. Service Package를 설치했다면 Day 6에서 배운 어떤 명령들을 연결해야 하는가?

---

## 다음 학습

**Day 7-4 — Package 업데이트, Version, hold/unhold, upgrade/full-upgrade**

다음에는:

```text
apt list --upgradable
apt policy
apt upgrade
apt full-upgrade
Package Version
hold / unhold
업데이트 영향도 확인
```

을 연결해서 학습한다.
