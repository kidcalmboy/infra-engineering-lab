# Day 7-2 — APT Repository 구조와 `apt update` 내부 동작

> APT가 어떤 Repository를 바라보는지, `apt update`가 실제로 어떤 정보를 받아 어디에 저장하는지, 그리고 그 정보가 이후 `apt install`과 어떻게 연결되는지 정리한다.

## 📌 이번에 배운 내용

- APT Repository 설정 위치
- `/etc/apt/sources.list`
- `/etc/apt/sources.list.d/`
- 최근 Ubuntu에서 사용하는 `.sources` 형식
- Repository 항목의 `deb`, URL, Suite/Distribution, Component 의미
- `main`, `universe`, `restricted`, `multiverse`
- `apt update`의 내부 흐름
- Package Metadata / Package Index
- `Packages`, `Release`, `InRelease`
- Signature와 Checksum이 필요한 이유
- `Hit`, `Get`, `Ign`, `Err` 의미
- `/var/lib/apt/lists/`
- `/var/cache/apt/archives/`
- 로컬 Package Index와 실제 `.deb` 캐시의 차이
- 외부 Repository와 신뢰 범위
- `apt update` 실패 시 기본 Troubleshooting 흐름

## 📚 목차

1. 전체 동작 구조
2. APT Repository 설정 파일
3. Repository 한 줄 해석
4. Suite와 Component
5. `apt update` 내부 동작
6. Package Metadata
7. `Packages`, `Release`, `InRelease`
8. Signature와 Checksum
9. `apt update` 출력 해석
10. 로컬 저장 위치
11. `apt install`과의 연결
12. 외부 Repository
13. `apt update` 장애 원인
14. 실무 운영 관점
15. 핵심 정리
16. 복습 문제

---

# 1. 전체 동작 구조

APT 패키지 관리는 다음 흐름으로 이해할 수 있다.

```text
Repository 설정
↓
apt가 사용할 저장소 결정
↓
apt update
↓
Repository의 최신 Metadata / Package Index 다운로드
↓
로컬에 저장
↓
apt search / apt policy / apt install이 이 정보를 사용
```

즉:

> **`apt update`는 실제 프로그램을 업데이트하는 명령이 아니라, Repository의 최신 패키지 정보를 로컬에 동기화하는 명령이다.**

---

# 2. APT Repository 설정 파일

대표 위치:

```text
/etc/apt/sources.list
/etc/apt/sources.list.d/
```

## `/etc/apt/sources.list`

전통적인 Repository 설정 파일이다.

예전 형식 예:

```text
deb http://archive.ubuntu.com/ubuntu noble main universe
```

## `/etc/apt/sources.list.d/`

Repository 설정을 파일별로 나눠 관리할 수 있는 디렉터리다.

예:

```text
/etc/apt/sources.list.d/ubuntu.sources
/etc/apt/sources.list.d/docker.sources
/etc/apt/sources.list.d/vendor.list
```

최근 Ubuntu에서는 `.list`뿐 아니라 `.sources` 형식이 사용될 수 있다.

따라서:

```text
sources.list만 확인
≠
전체 Repository 설정 확인
```

이다.

---

# 3. Repository 한 줄 해석

예:

```text
deb http://archive.ubuntu.com/ubuntu noble main universe
```

이를 나누면:

```text
deb
→ binary Debian Package Repository

http://archive.ubuntu.com/ubuntu
→ Repository 주소

noble
→ Distribution / Suite

main universe
→ Component
```

---

# 4. Suite / Distribution

예:

```text
noble
jammy
```

Ubuntu Release 코드네임과 연결된다.

예:

```text
Ubuntu 22.04 → jammy
Ubuntu 24.04 → noble
```

이 값은 대략:

> **어느 Ubuntu Release용 패키지 묶음을 사용할지**

정하는 역할을 한다.

---

# 5. Component

대표적인 Ubuntu Repository Component:

```text
main
universe
restricted
multiverse
```

간단한 구분:

| Component | 개념 |
|---|---|
| `main` | Ubuntu가 공식 지원하는 핵심 패키지 중심 |
| `universe` | 커뮤니티 유지 패키지 중심 |
| `restricted` | 제한된 라이선스/드라이버 계열 |
| `multiverse` | 라이선스 제약이 더 있는 소프트웨어 |

처음에는 Component를:

> **Repository 안에서 패키지를 분류한 영역**

으로 이해하면 된다.

---

# 6. `apt update` 내부 동작

```bash
sudo apt update
```

개념적 흐름:

```text
1. APT Repository 설정 읽기
2. 각 Repository 서버에 접속
3. 최신 Metadata 확인
4. 필요한 Package Index 다운로드
5. Signature / Checksum 검증
6. 로컬 Package List 갱신
7. 설치 가능 Version / Candidate / Upgrade 가능 여부 재계산
```

즉:

```text
원격 Repository 상태
↓
apt update
↓
로컬 Package Index 상태 갱신
```

이다.

---

# 7. Package Metadata란?

## 한 줄 정의

**Metadata는 패키지 파일 자체가 아니라 그 패키지를 설명하는 정보다.**

예:

```text
Package Name
Version
Architecture
Dependency
Description
Download Location
File Size
Checksum
```

APT는 이 정보를 보고:

```text
어떤 패키지가 존재하는가
어떤 버전이 있는가
무엇에 의존하는가
어디서 다운로드해야 하는가
```

를 판단한다.

---

# 8. `Packages` Index

Repository에는 패키지 목록을 담은 Index가 있다.

개념적인 내용:

```text
Package: nginx
Version: ...
Architecture: amd64
Depends: ...
Description: ...
Filename: ...
```

이런 항목이 여러 패키지에 대해 반복된다.

APT는 이 정보를 바탕으로 패키지 검색과 설치 후보를 판단한다.

---

# 9. `Release` / `InRelease`

Repository에는 Package 목록뿐 아니라 Repository 자체에 대한 Metadata도 존재한다.

대표적으로:

```text
Release
InRelease
```

등이 있다.

이런 Metadata에는 다음 정보가 포함될 수 있다.

```text
Suite
Codename
Component
Architecture
Checksum
Repository Metadata
```

`InRelease`는 Repository Metadata와 서명 검증 과정에서 중요한 역할을 한다.

---

# 10. 왜 Signature와 Checksum이 필요한가

패키지 관리에서는 단순히 파일을 내려받는 것보다 **신뢰성과 무결성**이 중요하다.

APT는 다음을 확인해야 한다.

```text
이 Repository Metadata는 신뢰 가능한 출처에서 왔는가?
전송 중 파일이 손상되거나 변조되지 않았는가?
```

---

## Checksum

**파일 내용이 바뀌었는지 검증하기 위한 요약값**이다.

개념:

```text
Expected Checksum
vs
Downloaded File Checksum
```

값이 다르면 파일 손상이나 변조 가능성을 의심할 수 있다.

---

## Digital Signature

Repository 제공자가 Metadata에 디지털 서명을 하고, 시스템은 신뢰하는 Public Key로 이를 검증한다.

```text
Repository
↓
Metadata 서명
↓
Ubuntu
↓
신뢰 Key로 Signature 검증
```

목적:

> **이 Metadata가 신뢰하는 Repository 제공자로부터 왔는지 확인**

---

# 11. `apt update` 출력 해석

실행:

```bash
sudo apt update
```

출력에서 볼 수 있는 대표 상태:

| 표시 | 의미 |
|---|---|
| `Hit` | 기존 로컬 정보가 유효하거나 새로 받을 내용이 없음 |
| `Get` | Repository에서 새 정보를 다운로드 |
| `Ign` | 특정 항목을 무시 |
| `Err` | Repository 접근/검증 등의 오류 |

정확한 출력은 Ubuntu/APT 버전과 Repository 상태에 따라 다를 수 있다.

---

# 12. Package Index는 어디에 저장되는가

APT가 Repository에서 받아온 Package Index는 보통:

```text
/var/lib/apt/lists/
```

아래에 저장된다.

확인:

```bash
ls -lh /var/lib/apt/lists/ | head -20
```

이 디렉터리는:

```text
Repository의 최신 패키지 정보
→ 로컬에 저장된 위치
```

라고 이해하면 된다.

---

# 13. 실제 `.deb` Package 파일은 어디에 저장되는가

APT가 실제 패키지를 다운로드하면 환경과 설정에 따라 캐시로:

```text
/var/cache/apt/archives/
```

를 사용할 수 있다.

두 경로를 반드시 구분한다.

```text
/var/lib/apt/lists/
→ Package Index / Metadata

/var/cache/apt/archives/
→ 실제 다운로드된 .deb Package 파일 캐시
```

즉:

```text
패키지 정보
≠
실제 패키지 파일
```

이다.

---

# 14. `apt update` 후 `apt policy`가 달라질 수 있는 이유

예를 들어 로컬 Index에는:

```text
nginx 1.24
```

만 기록돼 있는데 Repository에는 새 버전이 올라왔다고 가정한다.

```text
Repository: nginx 1.26
```

`apt update` 전에는 로컬 정보가 오래되어:

```text
Candidate: 1.24
```

처럼 보일 수 있다.

갱신 후:

```text
Repository
↓
apt update
↓
Local Index 최신화
↓
Candidate 재계산
↓
Candidate: 1.26
```

처럼 바뀔 수 있다.

실제 Version은 환경에 따라 다르며 위 숫자는 개념 예시다.

---

# 15. `apt install`과 연결

```bash
sudo apt install nginx
```

APT는 먼저 로컬 Package Index를 사용한다.

```text
nginx Package 정보 확인
↓
Candidate Version 결정
↓
Dependency 계산
↓
Repository 위치 확인
↓
실제 .deb 다운로드
↓
dpkg 계층으로 설치
```

따라서:

```text
apt update
→ 카탈로그 최신화

apt install
→ 카탈로그를 보고 실제 Package 다운로드 + 설치
```

라고 연결하면 된다.

---

# 16. APT 관련 주요 디렉터리

```text
/etc/apt/
→ APT 전체 설정

/etc/apt/sources.list
/etc/apt/sources.list.d/
→ Repository 설정

/var/lib/apt/lists/
→ Repository에서 받은 Package Index / Metadata

/var/cache/apt/archives/
→ 다운로드한 .deb Package 캐시 가능 위치
```

이 네 영역의 역할을 구분할 수 있어야 한다.

---

# 17. 외부 Repository

Ubuntu 공식 Repository 외에 Vendor가 별도 Repository를 제공할 수 있다.

예:

```text
Docker
PostgreSQL
Google Chrome
NodeSource
```

개념적인 흐름:

```text
외부 Repository 설정 추가
↓
apt update
↓
외부 Repository Metadata도 로컬 Index에 반영
↓
apt install
```

---

# 18. 외부 Repository가 왜 위험할 수 있는가

Repository를 추가한다는 것은 단순히 URL 하나를 추가하는 일이 아니다.

실질적으로는:

> **그 Repository에서 제공하는 패키지를 시스템에 설치할 수 있도록 신뢰 범위를 넓히는 것**

이다.

따라서 확인해야 할 것:

```text
공식 Vendor인가?
Repository URL이 정확한가?
지원하는 Ubuntu Release인가?
서명 Key는 신뢰할 수 있는가?
패키지 출처가 명확한가?
```

운영 서버에서는 특히 중요하다.

---

# 19. `apt update`가 Root 권한을 요구하는 이유

```bash
sudo apt update
```

를 사용하는 이유는 시스템 전역 Package Index를 갱신하기 때문이다.

대표적으로:

```text
/var/lib/apt/lists/
```

같은 시스템 관리 영역에 변경이 발생한다.

일반 사용자가 시스템 전체의 패키지 인덱스를 임의로 수정하지 못하게 Root 권한이 필요하다.

---

# 20. `apt update`와 네트워크

`apt update`는 원격 Repository에 접속하므로 기본적으로 네트워크 연결이 필요하다.

반면:

```bash
apt list --installed
```

같은 명령은 로컬 정보만으로 실행할 수 있다.

즉:

```text
원격 Repository 정보 필요
→ Network 필요

현재 설치된 Package 정보 확인
→ Local DB만으로 가능
```

---

# 21. `apt update` 실패 원인

대표 원인:

```text
Network 문제
DNS 문제
Repository URL 오류
지원 종료된 Ubuntu Release
GPG / Signature 문제
TLS / Certificate 문제
Proxy 문제
Repository Server 장애
```

---

# 22. 🔧 Troubleshooting

## 증상

```text
sudo apt update
→ Err 발생
```

## 접근

```text
1. 어떤 Repository에서 실패했는지 확인
2. Error Message 확인
3. Network 연결 확인
4. DNS Resolution 확인
5. sources.list / sources.list.d 확인
6. 시스템 시간 확인
7. Signature / GPG 오류 여부 확인
8. 해당 Ubuntu Release 지원 여부 확인
9. Repository Server 자체 장애 가능성 확인
```

예를 들어:

```text
Temporary failure resolving ...
→ DNS 가능성

404 Not Found
→ Repository URL / Suite / 지원 종료 여부 가능성

NO_PUBKEY / Signature Error
→ Repository Signing Key 관련 가능성
```

처럼 **오류 메시지를 근거로 가설을 좁힌다.**

---

# 23. 💼 실무 운영 관점

Package Repository 문제도 기존 Troubleshooting 원칙과 동일하다.

잘못된 흐름:

```text
apt update 실패
→ Repository 파일 아무거나 수정
→ Key 다시 추가
→ 재부팅
```

권장 흐름:

```text
오류 Repository 확인
→ Error Message 해석
→ Network / DNS / URL / Signature 가설 수립
→ 설정 확인
→ 최소 변경
→ apt update 재실행
→ 정상 Repository 상태 검증
```

또한 외부 Repository를 추가할 때는:

```text
출처 확인
→ 지원 OS Version 확인
→ Signing 방법 확인
→ Repository 설정 추가
→ apt update
→ apt policy로 출처/Version 확인
→ 필요한 Package 설치
```

순서가 안전하다.

---

# 24. 🧪 Day 7-2 관찰 실습

Repository 설정 확인:

```bash
cat /etc/apt/sources.list
```

최근 Ubuntu에서 별도 `.sources` 파일을 사용하는 경우:

```bash
ls -l /etc/apt/sources.list.d/
cat /etc/apt/sources.list.d/ubuntu.sources
```

Package Index 확인:

```bash
ls -lh /var/lib/apt/lists/ | head -20
```

Package Cache 확인:

```bash
ls -lh /var/cache/apt/archives/
```

현재 Package Candidate 확인:

```bash
apt policy openssh-client
```

Package Index 갱신:

```bash
sudo apt update
```

다시 확인:

```bash
apt policy openssh-client
```

### 관찰 포인트

```text
Repository URL
Ubuntu Suite / Codename
Component
Hit / Get / Ign / Err
/var/lib/apt/lists/ 파일
Installed / Candidate
```

실제 출력 원문이 공유되지 않은 경우 특정 Repository URL이나 Version을 사용자의 환경값처럼 기록하지 않는다.

---

# 25. ✅ 핵심 정리

APT 구조:

```text
/etc/apt/sources.list
/etc/apt/sources.list.d/
        ↓
"어떤 Repository를 사용할 것인가"

Repository Server
        ↓
Release / InRelease / Packages
        ↓
Signature / Checksum 검증
        ↓
apt update
        ↓
/var/lib/apt/lists/
        ↓
Local Package Index 최신화
        ↓
apt search / apt policy / apt install
```

설치 시:

```text
apt install PACKAGE
↓
Local Package Index 확인
↓
Candidate 선택
↓
Dependency 계산
↓
Repository에서 실제 .deb 다운로드
↓
/var/cache/apt/archives/ 사용 가능
↓
dpkg 계층으로 설치
```

가장 중요한 구분:

```text
Repository
→ 실제 Package + Metadata 제공

/var/lib/apt/lists/
→ 로컬 Package Index / Metadata

/var/cache/apt/archives/
→ 실제 .deb Package 캐시

apt update
→ Metadata 최신화

apt install
→ Metadata를 참고해 실제 Package 다운로드 + 설치
```

---

# 26. 🧠 복습 문제

1. APT Repository 설정은 대표적으로 어느 경로에서 확인하는가?
2. `sources.list`와 `sources.list.d/`의 차이는 무엇인가?
3. Repository 설정의 `deb`는 무엇을 의미하는가?
4. Suite/Distribution은 무엇을 지정하는가?
5. `main`, `universe` 등은 무엇인가?
6. `apt update`는 실제로 어떤 작업을 수행하는가?
7. Package Metadata에는 어떤 정보가 들어갈 수 있는가?
8. `Packages` Index는 어떤 역할을 하는가?
9. `Release` / `InRelease`는 왜 필요한가?
10. Checksum과 Digital Signature의 역할 차이는 무엇인가?
11. `Hit`, `Get`, `Err`는 각각 어떤 의미인가?
12. `/var/lib/apt/lists/`와 `/var/cache/apt/archives/`의 차이는?
13. `apt update` 후 `apt policy`의 Candidate가 달라질 수 있는 이유는?
14. 외부 Repository를 추가할 때 신뢰성을 확인해야 하는 이유는?
15. `apt update` 실패 시 어떤 순서로 원인을 좁혀야 하는가?

---

## 다음 학습

**Day 7-3 — 실제 Package 설치/삭제와 dpkg 상태 확인**

다음에는 안전한 Package 하나를 대상으로:

```text
apt install
→ dpkg -l
→ dpkg -L
→ dpkg -S
→ apt remove
→ 설정 파일 상태 확인
→ apt purge
→ autoremove
→ 설치/삭제 후 검증
```

흐름을 직접 실습한다.
