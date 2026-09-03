# Day 3-1 - Linux 사용자·그룹·sudo·su

> Linux에서 사용자를 UID/GID로 식별하는 방식과 사용자·그룹 관리, `sudo`, `su`의 차이를 실습했다.

## 📌 이번에 배운 내용

- User / Group / UID / GID
- `/etc/passwd`, `/etc/shadow`, `/etc/group`
- `whoami`, `id`, `groups`
- `adduser`, `passwd`, `groupadd`, `usermod -aG`
- `sudo` 권한과 최소 권한 원칙
- `su username`과 `su - username`의 차이
- 그룹 변경이 기존 로그인 세션에 바로 반영되지 않는 이유

## 📚 목차

1. 사용자와 그룹
2. UID와 GID
3. 계정 관련 파일
4. 사용자·그룹 관리 명령어
5. `sudo`
6. `su`와 로그인 환경
7. 실습
8. 헷갈리기 쉬운 부분
9. Troubleshooting
10. 실무 포인트
11. 핵심 정리
12. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 용도 |
|---|---|
| `whoami` | 현재 실행 사용자 확인 |
| `id` | UID, GID, 그룹 확인 |
| `groups` | 현재 사용자의 그룹 확인 |
| `id testuser` | 특정 사용자 계정 확인 |
| `grep "^testuser:" /etc/passwd` | 특정 계정 정보 확인 |
| `ls -ld /home/testuser` | 홈 디렉터리 자체 정보 확인 |
| `sudo adduser testuser` | Ubuntu에서 사용자 생성 |
| `passwd` | 현재 사용자 비밀번호 변경 |
| `sudo passwd testuser` | 관리자가 특정 사용자 비밀번호 변경 |
| `sudo groupadd ops` | 그룹 생성 |
| `sudo usermod -aG ops testuser` | 기존 보조 그룹을 유지하면서 `ops` 추가 |
| `su testuser` | 현재 환경 일부를 유지하며 사용자 전환 |
| `su - testuser` | 대상 사용자의 로그인 환경으로 전환 |
| `exit` | 전환한 셸 종료 |

---

## 1. 사용자와 그룹

### 한 줄 정의

**User**는 시스템을 사용하는 주체이고, **Group**은 여러 사용자에게 공통 권한을 적용하기 위한 묶음이다.

### 상세 설명

Linux 서버는 여러 사용자가 동시에 사용할 수 있다. 모든 사용자가 같은 권한을 가지면 시스템 파일이나 서비스 설정을 불필요하게 수정할 수 있으므로, 사용자와 그룹을 기준으로 접근을 제어한다.

```text
User + Group + Permission
```

이 구조는 다음 Day에서 배우는 파일 권한과 직접 연결된다.

### 실무에서 중요한 이유

운영 서버에서는 업무 역할에 따라 그룹을 만들고 필요한 사용자만 해당 그룹에 추가하는 방식이 자주 사용된다. 사용자마다 모든 권한을 개별 설정하는 것보다 관리가 쉽고 일관성을 유지하기 좋다.

---

## 2. UID와 GID

### 한 줄 정의

Linux는 사용자와 그룹을 이름이 아니라 내부적으로 **숫자 ID**로 식별한다.

```text
UID = User ID
GID = Group ID
```

확인:

```bash
id
```

예:

```text
uid=1000(linuxuser) gid=1000(linuxuser) groups=1000(linuxuser),27(sudo)
```

`linuxuser`라는 이름은 사람이 읽기 쉽게 보여주는 표현이고, 실제 권한과 소유권 판단에서는 UID/GID가 핵심이다.

---

## 3. 계정 관련 파일

### `/etc/passwd`

사용자 계정의 기본 정보를 저장한다.

```bash
grep "^testuser:" /etc/passwd
```

예:

```text
testuser:x:1001:1001:,,,:/home/testuser:/bin/bash
```

구조:

```text
username : x : UID : GID : comment : home : shell
```

`x`는 비밀번호 원문이 이 파일에 있다는 뜻이 아니라, 비밀번호 인증 정보가 `/etc/shadow`에 분리되어 있다는 의미다.

### `/etc/shadow`

비밀번호 해시와 비밀번호 정책 관련 정보를 저장하는 보호 파일이다.

```bash
sudo grep "^testuser:" /etc/shadow
```

일반 사용자가 읽지 못하도록 권한이 제한된다.

### `/etc/group`

그룹 정보를 저장한다.

```bash
grep "^ops:" /etc/group
```

예:

```text
ops:x:1002:testuser
```

```text
그룹명 : x : GID : 보조 그룹 구성원
```

---

## 4. 사용자·그룹 관리 명령어

### `adduser`

Ubuntu에서 사용자를 생성할 때 편리한 대화형 명령이다.

```bash
sudo adduser testuser
```

사용자 생성 후에는 반드시 확인한다.

```bash
id testuser
grep "^testuser:" /etc/passwd
ls -ld /home/testuser
```

### `passwd`

```bash
passwd
```

현재 사용자의 비밀번호를 변경한다.

```bash
sudo passwd testuser
```

관리자가 `testuser`의 비밀번호를 변경한다.

### `groupadd`

```bash
sudo groupadd ops
```

새 그룹을 생성한다.

### `usermod -aG`

```bash
sudo usermod -aG ops testuser
```

옵션:

| 옵션 | 의미 |
|---|---|
| `-G` | supplementary groups, 보조 그룹 목록 지정 |
| `-a` | append, 기존 보조 그룹을 유지하면서 추가 |

> 그룹을 **추가**할 때는 `-aG`를 함께 사용하는 습관이 중요하다. `-G`만 사용하면 기존 보조 그룹 목록이 교체될 수 있다.

---

## 5. `sudo`

### 한 줄 정의

`sudo`는 정책에 의해 허가된 사용자가 **특정 명령을 다른 사용자 권한으로 실행**하게 하는 도구다. 기본 대상 사용자는 일반적으로 root다.

예:

```bash
sudo apt update
```

현재 로그인 계정이 영구적으로 root로 바뀌는 것이 아니라 해당 명령만 상승된 권한으로 실행된다.

### 모든 사용자가 `sudo`를 사용할 수 있는가?

아니다.

Ubuntu에서는 보통 `sudo` 그룹 구성원이 관리자 명령을 사용할 수 있도록 설정되어 있지만, 최종 권한은 `/etc/sudoers`와 `/etc/sudoers.d/` 정책으로 결정된다.

확인:

```bash
groups linuxuser
groups testuser
```

### 실무에서 중요한 이유

관리 작업을 위해 항상 root로 로그인하는 것보다 평소에는 일반 계정을 사용하고 필요한 명령만 `sudo`로 실행하는 것이 안전하다.

이 원칙은 **Principle of Least Privilege, 최소 권한 원칙**과 연결된다.

---

## 6. `su`와 로그인 환경

### `su testuser`

```bash
su testuser
```

실행 사용자는 `testuser`로 바뀌지만 현재 작업 디렉터리와 일부 환경은 기존 셸의 값을 유지할 수 있다.

### `su - testuser`

```bash
su - testuser
```

대상 사용자가 새로 로그인한 것과 유사한 **login shell 환경**을 만든다. 일반적으로 대상 사용자의 홈 디렉터리로 이동하고 로그인용 환경 설정을 적용한다.

### 비교

| 구분 | `su testuser` | `su - testuser` |
|---|---|---|
| 실행 사용자 변경 | O | O |
| 현재 작업 디렉터리 유지 | 보통 O | 보통 X |
| 대상 사용자 홈으로 이동 | X | O |
| 로그인 환경 재구성 | X | O |
| 실제 로그인 환경 재현 | 제한적 | 더 적합 |

차이 확인:

```bash
whoami
pwd
echo $HOME
```

> 특정 사용자의 실제 로그인 환경을 재현해야 할 때는 일반적으로 `su - username`이 더 적합하다.

---

## 7. 🧪 실습

### 사용자 생성과 확인

```bash
sudo adduser testuser
id testuser
groups testuser
grep "^testuser:" /etc/passwd
ls -ld /home/testuser
```

### 그룹 생성과 사용자 추가

```bash
sudo groupadd ops
sudo usermod -aG ops testuser
groups testuser
```

실습에서는 다음처럼 `testuser`가 `ops` 그룹에 추가된 것을 확인했다.

```text
testuser : testuser users ops
```

### 사용자 전환

```bash
su - testuser
whoami
pwd
echo $HOME
exit
```

---

## 8. ⚠️ 헷갈리기 쉬운 부분

### `su testuser`를 하면 디렉터리만 기존 사용자 것이고 나머지는 모두 testuser인가?

정확하지 않다.

실행 UID는 `testuser`로 바뀌지만 **현재 작업 디렉터리뿐 아니라 일부 환경 변수와 셸 환경도 기존 환경을 유지할 수 있다.**

따라서 단순히 “디렉터리만 유지된다”고 기억하면 안 된다.

### `sudo`를 쓰면 root 계정으로 바뀌는가?

아니다. 보통 지정한 명령만 root 권한으로 실행한다.

### `/etc/passwd`에 비밀번호가 저장되는가?

현대 Linux에서는 실제 비밀번호 해시를 `/etc/shadow`에 저장한다. `/etc/passwd`의 `x`는 shadow 파일 사용을 나타낸다.

---

## 9. 🔧 Troubleshooting

### 문제 1 - 사용자를 만들었는데 홈 디렉터리가 없다고 한다

바로 `mkdir`를 실행하지 않고 상태부터 확인한다.

```bash
id testuser
grep "^testuser:" /etc/passwd
ls -ld /home/testuser
```

확인 순서:

```text
계정 존재 여부
→ 설정된 홈 경로
→ 실제 디렉터리 존재 여부
→ 원인 판단
```

### 문제 2 - 그룹에 추가했는데 현재 세션에서 보이지 않는다

`usermod -aG`로 그룹을 변경해도 **이미 만들어진 로그인 세션의 그룹 정보가 자동으로 갱신되지 않을 수 있다.**

해결:

```bash
exit
su - testuser
groups
```

> `usermod`를 반복 실행하는 것이 아니라 **새 로그인 세션을 생성**해야 한다.

---

## 10. 💼 실무 포인트

계정·권한 문제는 바로 수정하기보다 다음 순서로 확인한다.

```text
누구로 실행 중인가? → whoami / id
어떤 그룹인가?      → groups
계정 설정은?         → /etc/passwd
실제 홈은?           → ls -ld
sudo 정책은?         → sudo 그룹 / sudoers
세션이 오래됐는가?   → 재로그인 후 확인
```

운영 장애에서는 **계정 정보와 현재 세션 정보가 다를 수 있다는 점**을 기억해야 한다.

---

## 11. ✅ 핵심 정리

- Linux는 사용자와 그룹을 내부적으로 UID/GID로 식별한다.
- `/etc/passwd`는 계정 기본 정보, `/etc/shadow`는 비밀번호 인증 정보, `/etc/group`은 그룹 정보를 저장한다.
- `usermod -aG`는 기존 보조 그룹을 유지하면서 그룹을 추가한다.
- `sudo`는 허가된 사용자가 특정 명령을 상승된 권한으로 실행하게 한다.
- `su testuser`는 사용자만 전환하는 데 가깝고 기존 환경 일부가 남을 수 있다.
- `su - testuser`는 대상 사용자의 로그인 환경을 재현하는 데 더 적합하다.
- 그룹 변경은 기존 로그인 세션에 즉시 반영되지 않을 수 있다.

---

## 12. 🧠 복습 문제

1. UID와 GID는 각각 무엇을 의미하는가?
2. `/etc/passwd`, `/etc/shadow`, `/etc/group`의 역할은 무엇인가?
3. `usermod -aG ops testuser`에서 `-a`를 빼면 어떤 위험이 있는가?
4. 모든 일반 사용자가 `sudo`를 사용할 수 있는가?
5. `su testuser`와 `su - testuser`의 핵심 차이는 무엇인가?
6. 그룹 변경 후 기존 세션에 새 그룹이 안 보일 때 어떻게 해결해야 하는가?
