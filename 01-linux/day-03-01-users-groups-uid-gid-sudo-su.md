# Day 3-1 — Linux 사용자·그룹·`sudo`·`su`

> Linux가 사용자를 이름이 아니라 UID/GID로 식별하는 방식과 사용자·그룹·권한 위임 구조를 학습했다. 단순 계정 생성이 아니라 **Identity → Group Membership → File Permission → Privilege Escalation**이 어떻게 연결되는지 이해하는 것이 목표다.

## 📌 이번에 배운 내용

- User / Group / UID / GID
- Primary Group / Supplementary Group
- `/etc/passwd`, `/etc/shadow`, `/etc/group`
- NSS(Name Service Switch) 기초
- `whoami`, `id`, `groups`, `getent`
- `adduser`, `useradd`, `passwd`, `groupadd`, `usermod -aG`
- `sudo`와 sudoers 정책
- `su`와 `su -`
- 현재 세션과 그룹 정보 반영 시점

## 📚 목차

1. Linux의 사용자 Identity
2. UID와 GID
3. Primary/Supplementary Group
4. 계정 정보 파일
5. NSS와 `getent`
6. 사용자/그룹 관리 명령어
7. `sudo`
8. `su`와 login shell
9. 실제 실습
10. 헷갈리기 쉬운 부분
11. Troubleshooting
12. 실무 포인트
13. 핵심 정리
14. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 옵션/인자 | 의미 |
|---|---|---|
| `whoami` | 없음 | 현재 effective user 이름 확인 |
| `id` | 사용자 생략 시 현재 사용자 | UID/GID/그룹 확인 |
| `id testuser` | 사용자명 | 특정 사용자 Identity 확인 |
| `groups testuser` | 사용자명 | 소속 그룹 확인 |
| `getent passwd testuser` | DB=`passwd` | NSS 경로로 사용자 조회 |
| `getent group ops` | DB=`group` | NSS 경로로 그룹 조회 |
| `adduser testuser` | 사용자명 | Ubuntu/Debian 계열 사용자 생성 도우미 |
| `useradd -m testuser` | `-m` = make home | 낮은 수준 사용자 생성 |
| `groupadd ops` | 그룹명 | 그룹 생성 |
| `usermod -aG ops testuser` | `-a` append, `-G` supplementary groups | 기존 보조 그룹 유지하며 추가 |
| `sudo -l` | `-l` = list | 현재 sudo 허용 정책 확인 |
| `su - testuser` | `-` = login environment | 대상 사용자의 로그인 환경으로 전환 |

---

## 1. Linux의 사용자 Identity

Linux는 Multi-user OS다. 여러 사용자와 서비스 계정이 동시에 Process를 실행하므로 **누가 어떤 자원에 접근할 수 있는지** 구분해야 한다.

사용자 이름은 사람이 보기 편한 label이고, Kernel은 권한 판단에 숫자 Identity를 사용한다.

```text
username ↔ UID
 groupname ↔ GID
```

예:

```text
linuxuser ↔ UID 1000
testuser  ↔ UID 1001
```

---

## 2. UID와 GID

### UID — User ID

사용자를 식별하는 숫자다.

### GID — Group ID

그룹을 식별하는 숫자다.

확인:

```bash
id linuxuser
```

예:

```text
uid=1000(linuxuser) gid=1000(linuxuser) groups=1000(linuxuser),27(sudo),...
```

해석:

```text
uid=1000      → User ID
gid=1000      → Primary Group ID
groups=...    → 사용자가 속한 전체 그룹
```

### 왜 이름 대신 숫자를 쓰나?

파일 metadata나 Process credential에 이름 문자열 대신 UID/GID를 저장하면 Kernel이 일관되게 권한을 판단할 수 있다. 사용자 이름이 바뀌어도 UID가 같으면 내부 Identity는 유지될 수 있다.

---

## 3. Primary Group과 Supplementary Group

### Primary Group

사용자에게 기본으로 연결되는 대표 Group이다. 일반적으로 새 파일 생성 시 Group ownership 결정에 영향을 준다.

### Supplementary Group

사용자가 추가로 속한 Group들이다. 협업 디렉터리, 운영 권한, sudo 권한 등을 역할별로 부여할 때 사용한다.

예:

```text
testuser
Primary Group       = testuser
Supplementary Group = users, ops
```

### `usermod -aG`

```bash
sudo usermod -aG ops testuser
```

- `-G`: supplementary group list를 설정
- `-a`: append. 기존 list를 유지하면서 추가

> `-G`만 사용하면 기존 Supplementary Group 목록이 의도치 않게 교체될 수 있다. “추가”가 목적이면 `-aG` 조합을 기억한다.

---

## 4. 계정 정보 파일

### `/etc/passwd`

사용자 기본 정보가 저장된다.

```text
username:x:UID:GID:GECOS:home:shell
```

예:

```text
testuser:x:1001:1001:,,,:/home/testuser:/bin/bash
```

`x`는 실제 password가 여기 저장되어 있다는 뜻이 아니다. password hash는 보통 `/etc/shadow`에 있다.

### `/etc/shadow`

Password hash와 password aging 정책 정보를 저장한다. 일반 사용자 읽기 권한이 제한된다.

### `/etc/group`

Group 정보가 저장된다.

```text
groupname:x:GID:member1,member2
```

중요: 마지막 member list는 **명시적인 supplementary membership** 중심으로 보일 수 있어서 primary membership이 거기에 안 나타날 수 있다.

---

## 5. NSS와 `getent`

### NSS란?

NSS(Name Service Switch)는 사용자·그룹·호스트 이름 등의 정보를 어디에서 조회할지 정하는 glibc 기반 조회 체계다.

로컬 파일만 쓰는 시스템에서는 `/etc/passwd`, `/etc/group`이 핵심이지만, 기업 환경에서는 LDAP 같은 외부 Identity Source를 사용할 수도 있다.

그래서:

```bash
grep '^testuser:' /etc/passwd
```

은 로컬 파일만 직접 보는 반면:

```bash
getent passwd testuser
```

는 시스템이 실제로 사용하는 NSS 조회 경로를 따른다.

실무에서는 `getent`가 더 범용적이다.

---

## 6. 사용자/그룹 관리 명령어

### `adduser`

Ubuntu/Debian에서 사용자 생성 과정을 편하게 처리하는 higher-level 도구다.

```bash
sudo adduser testuser
```

보통 홈 디렉터리 생성, password 설정, 사용자 전용 그룹 생성 등을 함께 처리한다.

### `useradd`

더 낮은 수준의 계정 생성 명령이다.

```bash
sudo useradd -m testuser
```

`-m` = home directory 생성.

> `adduser`와 `useradd`는 같은 명령의 별칭이 아니다.

### `passwd`

```bash
passwd
sudo passwd testuser
```

현재 사용자 또는 지정 사용자의 password를 변경한다.

### `groupadd`

```bash
sudo groupadd ops
```

새 Group을 생성한다.

---

## 7. `sudo`

### 한 줄 정의

`sudo`는 sudoers 정책이 허용하는 범위에서 **다른 사용자 권한으로 명령 하나를 실행**하게 하는 도구다. 기본 target user는 root인 경우가 많다.

```bash
sudo systemctl status ssh
```

현재 Shell이 영구적으로 root로 변하는 것이 아니다.

### `sudo -l`

```bash
sudo -l
```

현재 사용자가 어떤 명령을 sudo로 실행할 수 있는지 확인한다.

### sudoers

정책은 보통:

```text
/etc/sudoers
/etc/sudoers.d/
```

에서 관리된다.

Ubuntu에서는 `sudo` Group이 관리자 권한과 연결되는 기본 구성이 흔하지만, 실제 허용 범위는 sudoers 정책이 결정한다.

### 최소 권한 원칙

운영에서는 root Shell을 계속 사용하는 것보다 필요한 명령에만 sudo를 사용하는 것이 안전하다.

```text
내 권한으로 가능 → sudo 사용 안 함
관리 권한 필요   → 필요한 명령에만 sudo
```

---

## 8. `su`와 login shell

### `su testuser`

사용자 Identity를 전환하지만 현재 작업 디렉터리와 일부 환경이 남을 수 있다.

### `su - testuser`

`-`는 login shell 환경을 요청한다. 대상 사용자의 home과 login environment를 더 가깝게 재현한다.

비교:

| 항목 | `su testuser` | `su - testuser` |
|---|---|---|
| 사용자 전환 | O | O |
| 현재 cwd 유지 | 보통 O | 보통 대상 Home으로 이동 |
| login environment | 제한적 | 더 가깝게 재현 |

확인:

```bash
whoami
pwd
echo $HOME
```

---

## 9. 🧪 실제 실습

```bash
sudo adduser testuser
id testuser
groups testuser
getent passwd testuser
```

Group 생성/추가:

```bash
sudo groupadd ops
sudo usermod -aG ops testuser
id testuser
groups testuser
```

사용자 전환:

```bash
su - testuser
whoami
pwd
echo $HOME
exit
```

실제 `linuxuser`는 UID/GID 1000이며 `adm`, `sudo`, `users`, `lxd` 등의 그룹을 가지고 있음을 확인했다.

---

## 10. ⚠️ 헷갈리기 쉬운 부분

> ⚠️ User와 Group은 이름이 같아도 다른 객체다. `testuser` 사용자와 `testuser` 그룹은 별개다.

> ⚠️ `/etc/group`의 member list가 비어 있다고 사용자가 그 Group에 전혀 속하지 않는다고 단정하면 안 된다. Primary Group membership은 다르게 표현된다.

> ⚠️ `sudo`는 root 계정으로 로그인하는 것과 다르다.

> ⚠️ `usermod -aG` 후 이미 열린 login session의 group credential이 즉시 갱신되지 않을 수 있다.

---

## 11. 🔧 Troubleshooting

### 증상: 그룹에 추가했는데 현재 Shell에서 권한이 안 생김

확인:

```bash
id
groups
id testuser
```

원인:

```text
/etc/group 등 계정 DB는 변경됐지만
현재 login session이 이전 group credential을 유지하고 있음
```

조치:

```text
새 login session 생성
→ 다시 id/groups 확인
→ 실제 파일 접근 검증
```

무작정 `usermod`를 여러 번 반복하지 않는다.

---

## 12. 💼 실무 포인트

권한 문제 분석 흐름:

```text
현재 Identity 확인 → whoami / id
Group 확인          → id / groups / getent
계정 DB 확인        → getent passwd/group
파일 Owner/Group    → ls -l / ls -ld
sudo 정책           → sudo -l
Session freshness   → 재로그인 여부 확인
```

기업 환경에서는 LDAP/AD 연동 Linux 서버도 많기 때문에 `/etc/passwd`만 보는 습관보다 `getent`와 실제 effective credential 확인이 중요하다.

---

## 13. ✅ 핵심 정리

- Linux는 사용자/그룹을 내부적으로 UID/GID로 식별한다.
- Primary Group은 기본 Group, Supplementary Group은 추가 Role Group이다.
- `/etc/passwd`는 기본 계정 정보, `/etc/shadow`는 password hash/policy, `/etc/group`은 Group 정보다.
- `getent`는 NSS 경로를 통해 시스템이 실제 사용하는 Identity DB를 조회한다.
- `usermod -aG`는 기존 Supplementary Group을 유지하며 추가한다.
- `sudo`는 정책에 따른 privilege delegation이다.
- `su -`는 대상 사용자의 login environment 재현에 더 적합하다.

---

## 14. 🧠 복습 문제

1. Username과 UID의 차이는?
2. Primary Group과 Supplementary Group은 무엇이 다른가?
3. `/etc/passwd`, `/etc/shadow`, `/etc/group`의 역할은?
4. `getent`가 `grep /etc/passwd`보다 범용적인 이유는?
5. `usermod -aG`에서 `-a`가 중요한 이유는?
6. `sudo`와 root login은 어떻게 다른가?
7. `su user`와 `su - user`의 차이를 설명해보라.
8. 그룹 변경 후 새 세션이 필요한 이유는?
