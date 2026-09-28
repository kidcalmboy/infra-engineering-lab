# Day 7-1 — Linux 패키지 관리 기초: Package, Repository, apt, dpkg

> Ubuntu에서 소프트웨어가 어떤 단위로 배포되고, Repository와 로컬 패키지 정보가 어떤 관계를 가지며, `apt`와 `dpkg`가 각각 어떤 역할을 하는지 개념부터 정리한다.

## 📌 이번에 배운 내용

- Package가 무엇인지
- Program과 Package의 차이
- Debian/Ubuntu의 `.deb` 패키지 형식
- Dependency(의존성)의 의미
- Repository(저장소)의 역할
- 로컬에 저장되는 것은 패키지 전체가 아니라 주로 **패키지 목록/메타데이터**라는 점
- `apt update`와 `apt install`의 차이
- `apt`와 `dpkg`의 역할 차이
- `apt install/remove/purge/autoremove`
- `apt search/show/policy`
- `dpkg -l/-L/-S`
- 패키지 설치와 systemd Service의 연결
- 운영 서버에서 패키지 변경 전에 확인해야 할 사항

## 📚 목차

1. Package란 무엇인가
2. 왜 패키지 관리 시스템이 필요한가
3. `.deb` 패키지
4. Dependency
5. Repository
6. 로컬에 무엇이 저장되는가
7. `apt update`
8. `apt install`
9. `apt`와 `dpkg` 관계
10. 설치/삭제 관련 apt 명령
11. 패키지 조회 명령
12. `apt policy`
13. 패키지와 systemd 연결
14. 운영 관점
15. 헷갈리기 쉬운 부분
16. 핵심 정리
17. 복습 문제

---

# 1. Package란 무엇인가

## 한 줄 정의

**Package는 프로그램을 설치·업데이트·삭제·관리할 수 있도록 실행 파일, 설정 파일, 라이브러리 정보, 메타데이터 등을 묶어 놓은 배포 단위다.**

예를 들어 어떤 프로그램을 설치한다고 해서 실행 파일 하나만 복사되는 것은 아니다.

패키지에는 다음과 같은 요소가 포함될 수 있다.

```text
실행 파일
설정 파일
문서
라이브러리 관련 정보
버전
패키지 이름
아키텍처 정보
의존성 정보
설치/삭제 시 실행되는 스크립트
```

따라서:

```text
Program
≠
Package
```

이다.

### Program

실행 가능한 소프트웨어 자체에 가까운 개념.

### Package

Program을 시스템에 **배포하고 관리하기 위한 단위**.

---

# 2. 왜 패키지 관리 시스템이 필요한가

패키지 관리자가 없다면 프로그램을 설치할 때 사람이 직접 다음을 관리해야 한다.

```text
다운로드
→ 압축 해제
→ 컴파일
→ 실행 파일 복사
→ 설정 파일 배치
→ 필요한 라이브러리 설치
→ 버전 관리
→ 제거할 파일 추적
```

문제는 설치보다 **운영 이후 관리**다.

예:

```text
이 파일은 어느 프로그램이 설치했는가?
이 프로그램은 어떤 버전인가?
업데이트 가능한가?
어떤 라이브러리가 필요한가?
삭제할 때 무엇을 지워야 하는가?
```

패키지 관리 시스템은 이런 정보를 기록하고 관리한다.

---

# 3. Ubuntu의 패키지 형식 — `.deb`

Ubuntu는 Debian 계열 배포판이므로 대표적으로 Debian Package 형식인:

```text
.deb
```

를 사용한다.

예시 형태:

```text
nginx_1.24.0_amd64.deb
```

개념적으로 다음과 같은 정보를 표현할 수 있다.

```text
패키지 이름 : nginx
버전       : 1.24.0
아키텍처    : amd64
패키지 형식 : .deb
```

> 실제 파일명 규칙이나 버전 문자열은 패키지마다 다를 수 있다.

---

# 4. Dependency — 의존성

## 한 줄 정의

**Dependency는 어떤 패키지나 프로그램이 정상적으로 설치·실행되기 위해 필요한 다른 패키지나 라이브러리다.**

예:

```text
Application A
  ↓ requires
Library B
  ↓ requires
Library C
```

Application A만 받아도 B와 C가 없다면 정상 실행되지 않을 수 있다.

그래서 패키지 관리에서 중요한 작업이:

```text
무엇이 필요한가?
이미 설치되어 있는가?
추가로 어떤 패키지를 설치해야 하는가?
버전 조건은 맞는가?
```

를 계산하는 것이다.

---

# 5. Repository란?

## 한 줄 정의

**Repository는 패키지 파일과 패키지 메타데이터를 제공하는 소프트웨어 저장소다.**

Repository는 단순한 다운로드 폴더가 아니다.

대략 다음과 같은 정보를 제공한다.

```text
패키지 파일
패키지 이름
버전
Dependency
Architecture
설명
검증 관련 정보
```

쉽게 보면:

```text
APT Repository
= Ubuntu가 패키지를 가져오는 소프트웨어 창고
```

GitHub Repository와 같은 "Repository = 저장소"라는 단어를 사용하지만 저장하는 대상이 다르다.

```text
GitHub Repository
→ Source Code 저장소

APT Repository
→ Linux Package 저장소
```

---

# 6. 로컬에는 패키지가 전부 저장되어 있는가?

**아니다.**

이 부분이 이번 파트에서 가장 중요하다.

Ubuntu가 Repository에 존재하는 모든 `.deb` 파일을 미리 로컬에 저장하고 있는 것이 아니다.

로컬에는 주로 다음과 같은 **패키지 목록과 메타데이터**가 저장되어 있다.

예:

```text
nginx라는 패키지가 존재한다
현재 알려진 버전은 무엇이다
어떤 dependency가 필요하다
어느 Repository에서 받을 수 있다
어떤 Architecture용이다
```

즉:

```text
패키지 정보
→ 로컬에 저장 가능

실제 .deb 패키지 파일
→ 보통 필요할 때 Repository에서 다운로드
```

### 비유

```text
Repository
= 쇼핑몰 창고

로컬 Package Index
= 최신 상품 카탈로그

apt install
= 카탈로그를 보고 실제 상품을 주문해서 설치
```

---

# 7. `apt update`

```bash
sudo apt update
```

## 한 줄 정의

**설정된 Repository에서 최신 패키지 목록과 메타데이터를 받아 로컬 Package Index를 갱신한다.**

중요:

```text
apt update
≠
설치된 프로그램 자체 업데이트
```

흐름:

```text
Repository
↓
최신 Package Metadata
↓
apt update
↓
로컬 Package Index 갱신
```

이후 apt는 이 정보를 이용해서:

```text
어떤 패키지가 존재하는지
어떤 버전을 설치할 수 있는지
어떤 dependency가 필요한지
어떤 패키지가 업그레이드 가능한지
```

판단한다.

---

# 8. `apt install`

예:

```bash
sudo apt install nginx
```

## 한 줄 정의

**로컬 패키지 정보를 바탕으로 설치할 패키지와 Dependency를 결정하고, Repository에서 실제 패키지 파일을 다운로드하여 설치하는 명령이다.**

개념적 흐름:

```text
apt install nginx
↓
로컬 Package Index에서 nginx 정보 조회
↓
설치할 Version 결정
↓
Dependency 계산
↓
Repository에서 필요한 .deb 다운로드
↓
패키지 설치
↓
설치 상태 기록
```

따라서:

```text
apt update
→ "정보를 최신화"

apt install
→ "그 정보를 바탕으로 실제 패키지를 받아 설치"
```

이다.

---

# 9. `apt`와 `dpkg`의 관계

## `dpkg`

**Debian 패키지(`.deb`)를 직접 설치·삭제·조회하는 저수준 패키지 관리 도구**다.

예:

```bash
sudo dpkg -i package.deb
```

`-i`:

```text
install
```

### dpkg의 특징

```text
로컬 .deb 파일 직접 처리
설치된 Debian Package 정보 조회
패키지가 설치한 파일 조회
특정 파일이 어느 패키지 소속인지 조회
```

하지만 Repository에서 Dependency를 찾아 자동으로 해결하는 고수준 기능은 apt의 역할에 가깝다.

---

## `apt`

**Repository를 이용해 패키지를 검색하고, Dependency를 계산하며, 다운로드·설치·업데이트·삭제를 관리하는 고수준 도구**다.

개념적으로:

```text
사용자
↓
apt
↓
Repository / Dependency 계산
↓
필요한 .deb 확보
↓
dpkg 계층을 통해 패키지 설치/관리
```

라고 이해할 수 있다.

### 비교

| 항목 | apt | dpkg |
|---|---|---|
| 역할 | 고수준 패키지 관리 | 저수준 Debian Package 관리 |
| Repository 사용 | O | 직접 사용하지 않음 |
| Dependency 자동 해결 | O | 제한적 |
| `.deb` 직접 처리 | 가능 | 핵심 역할 |
| 설치/업데이트/검색 | 강함 | 제한적 |
| 로컬 Package DB 조회 | 가능 | 강함 |

간단히:

```text
apt
→ 패키지를 전체적인 관점에서 관리

dpkg
→ Debian Package 자체를 직접 처리
```

---

# 10. 설치/삭제 관련 apt 명령

## `apt upgrade`

```bash
sudo apt upgrade
```

현재 설치된 패키지들을 사용 가능한 새 버전으로 업그레이드한다.

보통:

```text
apt update
→ 최신 목록 확인

apt upgrade
→ 실제 설치 패키지 업그레이드
```

순서로 이해한다.

---

## `apt remove`

```bash
sudo apt remove nginx
```

패키지 제거 중심.

패키지가 관리하는 일부 설정 파일이 남을 수 있다.

---

## `apt purge`

```bash
sudo apt purge nginx
```

패키지와 패키지가 관리하는 설정 파일까지 제거하는 데 사용한다.

단:

> 사용자가 직접 만든 데이터, 홈 디렉터리 데이터 등 시스템의 모든 관련 파일을 자동으로 전부 삭제한다는 뜻은 아니다.

---

## `apt autoremove`

```bash
sudo apt autoremove
```

다른 패키지의 Dependency로 자동 설치되었지만 현재는 더 이상 필요하지 않은 패키지들을 제거하는 데 사용한다.

운영 서버에서는 제거 예정 목록을 먼저 확인해야 한다.

---

# 11. 패키지 조회 명령

## apt 버전

```bash
apt --version
```

이 명령은 패키지 버전을 묻는 것이 아니라:

> **현재 시스템에 설치된 apt 프로그램 자체의 버전**

을 확인한다.

예:

```text
apt 2.x.x (amd64)
```

---

## dpkg 버전

```bash
dpkg --version
```

현재 설치된 `dpkg` 프로그램 자체의 버전을 확인한다.

---

## 설치된 패키지 목록

```bash
apt list --installed
```

또는:

```bash
dpkg -l
```

특정 패키지 검색:

```bash
apt list --installed | grep nginx
dpkg -l | grep nginx
```

---

## 패키지 검색

```bash
apt search nginx
```

패키지 이름/설명 등을 기준으로 검색한다.

---

## 패키지 상세 정보

```bash
apt show nginx
```

예상 가능한 정보:

```text
Package
Version
Depends
Description
Installed-Size
```

---

## 특정 파일이 어느 패키지 소속인지

```bash
dpkg -S /usr/bin/ssh
```

`-S`는 특정 경로가 어떤 설치된 패키지에서 제공되었는지 찾을 때 사용한다.

예상 형태:

```text
openssh-client: /usr/bin/ssh
```

---

## 패키지가 설치한 파일 목록

```bash
dpkg -L openssh-client
```

`-L`:

```text
list files
```

즉:

> 이 패키지가 시스템에 어떤 파일들을 설치했는가?

를 확인한다.

---

# 12. `apt policy`

예:

```bash
apt policy openssh-client
```

패키지의 설치 상태와 사용 가능한 버전을 비교할 때 유용하다.

주요 개념:

```text
Installed
→ 현재 시스템에 설치된 버전

Candidate
→ apt가 현재 설치/업그레이드 대상으로 선택할 버전

Version table
→ 사용 가능한 Version과 Repository 관련 정보
```

이 명령을 통해:

```text
현재 설치된 버전
vs
Repository에서 설치 가능한 버전
```

을 비교할 수 있다.

---

# 13. Package와 systemd 연결

Day 6에서 배운 systemd와 패키지 관리가 연결된다.

예를 들어 어떤 Service Package를 설치하면 패키지는 다음을 함께 설치할 수 있다.

```text
실행 파일
설정 파일
systemd Unit 파일
문서
기타 리소스
```

예:

```text
apt install nginx
↓
nginx Package 설치
↓
nginx 실행 파일 / 설정 / nginx.service 등이 배치될 수 있음
↓
systemd가 Service 관리
```

하지만:

```text
Package 설치 완료
≠
Service가 반드시 정상 동작 중
```

이다.

설치 이후에도 필요하면:

```bash
systemctl status SERVICE
```

등으로 실제 Service 상태를 검증해야 한다.

---

# 14. 💼 운영 관점

운영 서버에서는 패키지 변경이 서비스에 영향을 줄 수 있다.

예:

```text
Web Server
Database
Runtime
Library
Kernel
```

업데이트가 다음으로 이어질 수 있다.

```text
설정 호환성 변화
Service restart
Process 교체
Connection 단절
Application 호환성 문제
재부팅 필요 가능성
```

따라서 무조건:

```bash
sudo apt upgrade
```

부터 실행하기보다는 먼저:

```bash
apt list --upgradable
```

등으로 변경 대상을 확인하는 습관이 중요하다.

운영 흐름 예:

```text
현재 Package/Version 확인
→ 업데이트 가능 목록 확인
→ 영향도 판단
→ 변경
→ Service 상태 확인
→ Log 확인
→ 실제 기능 검증
```

---

# 15. ⚠️ 헷갈리기 쉬운 부분

## Repository에 있는 패키지가 전부 내 서버에도 있는가?

아니다.

```text
Repository
→ 실제 Package 저장소

내 서버
→ 주로 Repository에서 받아둔 Package Index/Metadata 보유
```

필요한 Package 파일은 설치 시 다운로드한다.

---

## `apt update`는 프로그램 업데이트인가?

아니다.

```text
apt update
→ Package 목록/Metadata 갱신

apt upgrade
→ 설치된 Package 업그레이드
```

---

## `apt install`은 패키지 정보만 가져오는가?

아니다.

```text
로컬 Metadata 확인
→ 실제 Package 다운로드
→ Dependency 다운로드
→ 설치
```

까지 수행한다.

---

## `apt --version`은 Repository의 패키지 버전인가?

아니다.

```bash
apt --version
```

은 **apt 프로그램 자체의 버전**이다.

특정 패키지 버전을 보고 싶다면:

```bash
apt policy PACKAGE
```

또는 설치된 상태를 확인하는 다른 패키지 조회 명령을 사용한다.

---

## Package와 Repository 차이

```text
Package
→ 설치할 소프트웨어 단위

Repository
→ 여러 Package와 Metadata를 제공하는 저장소
```

---

## 로컬 Package Index와 dpkg Database 차이

이 둘도 구분해야 한다.

```text
APT Package Index
→ Repository에서 어떤 Package를 설치할 수 있는지에 대한 정보

dpkg Database
→ 현재 이 시스템에 어떤 Debian Package가 설치되어 있는지에 대한 정보
```

따라서:

```text
설치 가능한 패키지 정보
≠
현재 설치된 패키지 정보
```

이다.

---

# 16. ✅ 핵심 정리

전체 구조를 한 번에 보면:

```text
             ┌─────────────────────┐
             │   APT Repository    │
             │ .deb + Metadata     │
             └─────────┬───────────┘
                       │
                  apt update
                       │
                       ▼
             ┌─────────────────────┐
             │ Local Package Index │
             │ 이름/버전/의존성 등 │
             └─────────┬───────────┘
                       │
                 apt install
                       │
          ┌────────────┴────────────┐
          │                         │
       Dependency 계산        .deb Download
          │                         │
          └────────────┬────────────┘
                       ▼
                  Package 설치
                       │
                       ▼
                 dpkg Database
              "현재 무엇이 설치됐나"
```

가장 중요한 문장:

> **Repository에는 실제 패키지가 있고, 로컬에는 주로 그 패키지들의 목록/메타데이터가 저장되어 있으며, `apt install`은 그 정보를 이용해 실제 패키지를 Repository에서 다운로드하고 설치한다.**

또 하나:

```text
apt
→ Repository + Dependency + 설치/업데이트 관리

dpkg
→ Debian .deb Package 자체와 로컬 설치 상태 관리
```

---

# 17. 🧪 Day 7-1 기본 조회 실습

안전한 읽기 전용 실습:

```bash
apt --version
dpkg --version
apt policy
apt list --installed | head -20
dpkg -l | head -20
dpkg -S /usr/bin/ssh
dpkg -L openssh-client | head -20
apt policy openssh-client
```

확인 목표:

```text
apt / dpkg 자체 Version
설치된 Package 목록
특정 파일의 Package 소속
Package가 설치한 파일 목록
Installed / Candidate Version
```

실제 출력 원문을 공유하지 않은 경우 문서에는 특정 Version이나 결과를 임의로 기록하지 않는다.

---

# 18. 🧠 복습 문제

1. Program과 Package는 어떻게 다른가?
2. Ubuntu에서 주로 사용하는 Package 형식은 무엇인가?
3. Dependency란 무엇인가?
4. Repository는 무엇을 저장하는가?
5. Repository에 있는 모든 Package가 내 서버에 미리 다운로드되어 있는가?
6. 로컬 Package Index에는 무엇이 저장되는가?
7. `apt update`와 `apt upgrade`의 차이는?
8. `apt install nginx`를 실행하면 내부적으로 어떤 흐름이 일어나는가?
9. `apt`와 `dpkg`의 역할 차이는?
10. `dpkg -S /usr/bin/ssh`는 무엇을 확인하는 명령인가?
11. `dpkg -L openssh-client`는 무엇을 보여주는가?
12. `apt policy PACKAGE`의 Installed와 Candidate는 무엇인가?
13. `apt --version`은 어떤 버전을 보여주는가?
14. Package 설치와 Service 실행이 같은 의미가 아닌 이유는?
15. 운영 서버에서 `apt upgrade` 전에 변경 대상을 확인해야 하는 이유는?

---

## 다음 학습

**Day 7-2 — APT Repository 구조와 `apt update` 내부 흐름**

다음에는 다음 내용을 연결해서 학습한다.

```text
/etc/apt/sources.list
/etc/apt/sources.list.d/
Repository 설정
Package Index
apt update 실제 흐름
Cache
Repository Component / Architecture 기초
```
