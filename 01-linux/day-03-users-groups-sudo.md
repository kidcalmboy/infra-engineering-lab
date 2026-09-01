# Day 3 - 사용자, 그룹, sudo 기초

## 학습 목표

- Linux에서 사용자(User)와 그룹(Group)이 왜 필요한지 이해한다.
- `root`와 일반 사용자의 차이를 이해한다.
- UID와 GID의 의미를 이해한다.
- `/etc/passwd`, `/etc/shadow`, `/etc/group`의 역할을 구분한다.
- `whoami`, `id`, `groups`, `grep`, `ls -ld`를 이용해 계정 상태를 확인한다.
- `adduser`로 테스트 사용자를 생성하고 생성 결과를 검증한다.
- `sudo`가 모든 일반 사용자에게 허용되는 것이 아니라 권한이 부여된 사용자만 사용할 수 있다는 점을 이해한다.
- 문제 발생 시 바로 수정하기보다 계정 정보와 실제 파일시스템 상태를 먼저 확인하는 습관을 익힌다.

## 사용자와 그룹이 필요한 이유

Linux 서버에서는 여러 사용자가 동시에 시스템을 사용할 수 있다. 모든 사용자가 같은 권한을 가지면 시스템 설정이나 서비스에 불필요하게 접근할 수 있으므로, Linux는 사용자와 그룹을 기준으로 접근 권한을 관리한다.

```text
사용자(User)
+
그룹(Group)
+
권한(Permission)
```

사용자와 그룹 개념은 이후 파일 권한과 직접 연결된다.

## root와 일반 사용자

`root`는 Linux의 최고 관리자 계정이다. 일반 사용자는 평소 작업을 수행하고, 관리자 권한이 필요한 명령만 `sudo`를 통해 실행하는 방식이 일반적이다.

```text
평소 작업
→ 일반 사용자

관리 작업
→ sudo 명령어
```

현재 로그인한 사용자 확인:

```bash
whoami
```

## UID와 GID

Linux는 사용자를 이름만으로 관리하지 않고 숫자 ID로 식별한다.

- UID: User ID
- GID: Group ID

현재 계정의 UID, GID, 그룹 정보 확인:

```bash
id
```

예:

```text
uid=1000(linuxuser) gid=1000(linuxuser) groups=1000(linuxuser),27(sudo)
```

해석:

```text
uid=1000
→ linuxuser의 사용자 ID

gid=1000
→ 기본 그룹 ID

27(sudo)
→ sudo 그룹의 GID가 27이며 현재 사용자가 해당 그룹에 속함
```

특정 사용자 확인:

```bash
id testuser
```

Linux 내부에서는 파일 소유권과 프로세스 권한 등을 UID/GID 기준으로 처리한다.

## `groups`

현재 사용자가 속한 그룹 확인:

```bash
groups
```

특정 사용자의 그룹 확인:

```bash
groups testuser
```

실습에서는 `testuser`의 출력에 `sudo`가 없었기 때문에 sudo 권한이 없는 상태임을 확인했다.

## `/etc/passwd`

`/etc/passwd`는 사용자 계정 정보를 저장하는 파일이다.

특정 사용자 정보 확인:

```bash
grep "^testuser:" /etc/passwd
```

예:

```text
testuser:x:1001:1001:,,,:/home/testuser:/bin/bash
```

주요 필드:

```text
사용자명
x
UID
기본 GID
설명
홈 디렉터리
로그인 Shell
```

여기서 `x`는 실제 비밀번호 인증 정보가 `/etc/shadow`에 별도로 저장된다는 의미로 이해하면 된다.

## `^` 정규식 기호

다음 명령에서:

```bash
grep "^testuser:" /etc/passwd
```

`^`는 **줄의 시작**을 의미한다.

```text
^testuser:
→ 줄 맨 앞이 testuser:로 시작하는 줄만 검색
```

단순히 `grep "testuser"`를 사용하는 것보다 정확한 계정 검색에 유리하다.

## `/etc/shadow`

`/etc/shadow`는 비밀번호 인증과 관련된 보호 정보를 저장한다.

```bash
sudo grep "^testuser:" /etc/shadow
```

중요한 점:

```text
/etc/passwd
→ 사용자 계정 정보

/etc/shadow
→ 비밀번호 인증 관련 정보
```

`/etc/shadow`에 사용자의 실제 비밀번호 원문이 그대로 저장되는 것은 아니며, 해시 형태의 인증 정보가 저장된다.

## `/etc/group`

그룹 정보는 `/etc/group`에서 확인할 수 있다.

```bash
grep "^sudo:" /etc/group
```

예:

```text
sudo:x:27:linuxuser
```

대략적인 구조:

```text
그룹명 : x : GID : 그룹 구성원
```

## `ls -ld`

사용자의 홈 디렉터리 자체 정보를 확인할 때 사용한다.

```bash
ls -ld /home/testuser
```

옵션:

```text
-l
→ long format, 상세 정보 출력

-d
→ 디렉터리 안의 내용이 아니라 디렉터리 자체 정보 출력
```

예:

```text
drwxr-x--- 2 testuser testuser 4096 ... /home/testuser
```

현재 단계에서는 소유자와 그룹을 확인하는 데 집중하고, `drwxr-x---` 형태의 권한 표기는 이후 권한 관리 파트에서 자세히 다룬다.

## 사용자 생성 - `adduser`

Ubuntu에서 테스트 사용자를 생성했다.

```bash
sudo adduser testuser
```

사용자 생성 후 다음을 확인했다.

```bash
id testuser
groups testuser
grep "^testuser:" /etc/passwd
ls -ld /home/testuser
```

사용자를 만들면 사용자 이름만 생성되는 것이 아니라 UID, 기본 그룹, 홈 디렉터리, 로그인 Shell 등의 정보가 함께 구성된다.

## `adduser`와 `useradd`

Ubuntu에서 `adduser`는 사용자 생성 과정을 대화형으로 안내해주기 때문에 초보 실습에 편리하다.

```bash
sudo adduser testuser
```

`useradd`는 더 저수준 명령으로 옵션을 직접 지정하는 경우가 많다.

```bash
sudo useradd -m testuser2
```

현재 단계에서는 `adduser`를 기본으로 사용한다.

## `sudo`

`sudo`는 권한이 부여된 사용자가 특정 명령을 관리자(root) 권한으로 실행할 수 있게 해준다.

예:

```bash
sudo poweroff
sudo apt update
```

중요한 점은 `sudo`를 사용한다고 현재 사용자 계정 자체가 영구적으로 root가 되는 것은 아니라는 것이다.

```text
sudo poweroff
→ poweroff 명령만 관리자 권한으로 실행
```

## 모든 일반 사용자가 sudo를 사용할 수 있는 것은 아니다

일반 사용자라고 해서 자동으로 sudo 권한을 가지는 것은 아니다.

실습 환경에서는:

```text
linuxuser
→ sudo 그룹 포함
→ sudo 사용 가능

testuser
→ sudo 그룹 없음
→ 기본적으로 sudo 사용 불가
```

따라서 sudo 권한이 있는 계정은 강력한 시스템 작업을 수행할 수 있으므로 필요한 사용자에게만 권한을 부여해야 한다.

이와 연결되는 운영 원칙이 **최소 권한 원칙(Principle of Least Privilege)** 이다.

```text
업무 수행에 필요한 최소한의 권한만 부여한다.
```

sudo의 세부 권한 제어는 `/etc/sudoers`와 `/etc/sudoers.d/`를 이용해 관리할 수 있으며, 이후 `visudo`와 함께 다룬다.

## 실습 문제 확인

### 현재 사용자의 UID/GID 확인

```bash
id
```

### 현재 사용자가 속한 그룹 확인

```bash
groups
```

### testuser의 계정 정보 확인

```bash
grep "^testuser:" /etc/passwd
```

### testuser의 홈 디렉터리 확인

```bash
ls -ld /home/testuser
```

### testuser의 sudo 그룹 여부 확인

```bash
groups testuser
```

출력에 `sudo`가 없다면 sudo 그룹에 속해 있지 않은 상태다.

## Troubleshooting 실습

### 상황

`testuser` 계정을 만들었는데 `/home/testuser`가 존재하지 않는 것 같다는 제보가 들어왔다.

바로 디렉터리를 생성하지 않고 다음 순서로 사실을 확인한다.

```bash
id testuser
grep "^testuser:" /etc/passwd
ls -ld /home/testuser
```

해석:

```text
id testuser
→ 계정이 실제로 존재하는가?

grep "^testuser:" /etc/passwd
→ 계정의 홈 디렉터리가 어디로 설정되어 있는가?

ls -ld /home/testuser
→ 실제 디렉터리가 존재하는가?
```

핵심 Troubleshooting 흐름:

```text
증상 확인
→ 계정 존재 여부 확인
→ 계정 설정 확인
→ 실제 파일시스템 상태 확인
→ 원인 판단
→ 필요한 경우 조치
→ 검증
```

문제가 있다고 들었다고 해서 곧바로 `mkdir`나 설정 변경을 수행하기보다, 먼저 시스템이 실제로 어떤 상태인지 확인하는 것이 중요하다.

## 오늘까지 핵심 명령어

```bash
whoami
id
id testuser
groups
groups testuser
grep "^testuser:" /etc/passwd
grep "^sudo:" /etc/group
sudo grep "^testuser:" /etc/shadow
ls -ld /home/testuser
sudo adduser testuser
```

## 핵심 정리

```text
whoami
→ 현재 사용자

id
→ UID / GID / 그룹 확인

groups
→ 그룹 구성 확인

UID
→ 사용자 식별 번호

GID
→ 그룹 식별 번호

/etc/passwd
→ 사용자 계정 정보

/etc/shadow
→ 비밀번호 인증 관련 보호 정보

/etc/group
→ 그룹 정보

^
→ 줄의 시작

ls -ld
→ 디렉터리 자체 상세 정보

adduser
→ 사용자 생성

sudo
→ 권한이 부여된 사용자가 특정 명령을 관리자 권한으로 실행
```

## 다음 학습

다음 단계에서는 아래 내용을 실습한다.

```text
passwd
groupadd
usermod
사용자를 그룹에 추가/제거
su와 sudo 차이
userdel
sudo 권한 구조
```
