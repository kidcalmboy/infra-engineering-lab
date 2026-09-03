# Day 4-3 - 파일 소유권, 그룹과 `getent`

> Linux 파일의 Owner/Group이 무엇을 의미하는지 이해하고, `chown`, `chgrp`, `id`, `groups`, `getent`를 이용해 소유권과 그룹 정보를 확인·변경하는 방법을 학습했다.

## 📌 이번에 배운 내용

이번 학습에서는 파일 권한(`chmod`)과 파일 소유권(`chown`, `chgrp`)의 차이를 연결해서 학습했다.

주요 학습 내용:

- 파일의 Owner와 Group 의미
- `chown`으로 소유자 또는 소유자+그룹 변경
- `chgrp`로 그룹 소유권만 변경
- Primary Group과 Supplementary Group의 차이
- 사용자 이름과 같은 이름의 그룹이 존재할 수 있는 이유
- `id`, `groups`, `getent`를 이용한 사용자/그룹 확인
- 실제 `secret.conf` 파일의 그룹과 권한을 조정하는 실습
- 권한 문제에서 Owner/Group/`rwx`를 함께 확인해야 하는 이유

## 📚 목차

1. 파일 소유권이란
2. `chown`과 `chgrp`
3. 사용자와 그룹의 관계
4. Primary Group과 Supplementary Group
5. `getent`
6. 명령어 비교
7. 실습
8. 헷갈리기 쉬운 부분
9. Troubleshooting
10. 실무 포인트
11. 핵심 정리
12. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 의미 | 언제 사용하는가 |
|---|---|---|
| `ls -l file` | 파일 권한·Owner·Group 확인 | 현재 소유권과 권한 점검 |
| `chown user file` | Owner 변경 | 파일 소유자를 다른 사용자로 변경 |
| `chown user:group file` | Owner와 Group 동시 변경 | 소유 사용자와 소유 그룹을 함께 설정 |
| `chgrp group file` | Group만 변경 | Owner는 유지하고 그룹만 바꿀 때 |
| `id user` | UID, Primary GID, 전체 그룹 확인 | 사용자 권한/그룹 문제 점검 |
| `groups user` | 사용자가 속한 그룹 확인 | 그룹 멤버십 빠른 확인 |
| `getent passwd user` | 시스템 사용자 데이터베이스 조회 | 사용자 계정 정보 확인 |
| `getent group groupname` | 시스템 그룹 데이터베이스 조회 | 그룹의 GID와 멤버 확인 |
| `usermod -aG group user` | Supplementary Group 추가 | 기존 그룹을 유지하면서 새 그룹 추가 |

---

## 1. 파일 소유권이란

### 한 줄 정의

Linux 파일과 디렉터리는 **Owner(소유 사용자)**와 **Group(소유 그룹)**을 가지며, `chmod`의 Owner/Group 권한은 이 소유권 정보를 기준으로 적용된다.

### 상세 설명

예를 들어 다음 파일이 있다고 하자.

```text
-rw-r----- 1 linuxuser testuser 0 Sep 3 06:40 secret.conf
```

주요 부분은 다음과 같다.

```text
-rw-r-----   → 권한
linuxuser    → Owner
testuser     → Group
```

권한 `640`을 해석하면 다음과 같다.

```text
Owner  → rw-
Group  → r--
Others → ---
```

따라서 이 파일의 실제 접근 여부는 **권한 숫자만 보는 것이 아니라 Owner가 누구인지, Group이 무엇인지까지 함께 봐야 한다.**

### 실무에서 중요한 이유

운영 서버에서 `Permission denied`가 발생했을 때 단순히 `chmod`만 수정해서는 안 된다.

먼저 다음 세 가지를 함께 확인해야 한다.

```text
1. 파일 Owner는 누구인가?
2. 파일 Group은 무엇인가?
3. Owner / Group / Others에 어떤 rwx가 설정되어 있는가?
```

---

## 2. `chown`과 `chgrp`

### `chown`

### 한 줄 정의

`chown`은 **change owner**의 의미로 파일 또는 디렉터리의 소유자를 변경하는 명령어다.

```bash
sudo chown testuser file.txt
```

`file.txt`의 Owner를 `testuser`로 변경한다.

Owner와 Group을 동시에 변경할 수도 있다.

```bash
sudo chown testuser:ops file.txt
```

결과:

```text
Owner → testuser
Group → ops
```

### `chgrp`

### 한 줄 정의

`chgrp`는 **change group**의 의미로 파일 또는 디렉터리의 소유 그룹만 변경한다.

```bash
sudo chgrp ops file.txt
```

결과:

```text
Owner → 기존 값 유지
Group → ops
```

### `chmod`와의 차이

| 명령어 | 변경 대상 | 예시 |
|---|---|---|
| `chmod` | `rwx` 접근 권한 | `chmod 640 file` |
| `chown` | Owner 또는 Owner+Group | `chown user:group file` |
| `chgrp` | Group | `chgrp group file` |

> `chmod`는 **어떤 권한을 줄지**, `chown`/`chgrp`는 **그 권한을 적용받는 Owner/Group이 누구인지**를 결정한다.

---

## 3. 사용자와 그룹의 관계

### 한 줄 정의

Linux에서 **사용자(User)와 그룹(Group)은 서로 다른 객체**이며, 한 사용자는 하나 이상의 그룹에 속할 수 있다.

Ubuntu에서 `adduser testuser`처럼 사용자를 만들면 일반적으로 사용자와 같은 이름의 그룹도 함께 생성된다.

예:

```text
User  → testuser
Group → testuser
```

이름이 같아도 사용자와 그룹은 동일한 객체가 아니다.

이를 확인할 수 있다.

```bash
id testuser
```

```bash
getent group testuser
```

이 구조는 Ubuntu/Debian 계열에서 흔히 사용하는 **User Private Group(사용자 전용 그룹)** 방식과 관련 있다.

---

## 4. Primary Group과 Supplementary Group

### Primary Group

사용자에게 기본적으로 연결되는 주 그룹이다.

```bash
id testuser
```

예시:

```text
uid=1001(testuser) gid=1001(testuser) groups=1001(testuser),1002(ops)
```

여기서:

```text
uid=1001(testuser) → 사용자 UID
gid=1001(testuser) → Primary Group의 GID
groups=...         → 사용자가 속한 전체 그룹
```

사용자가 일반적으로 새 파일을 만들면 파일 Group은 사용자의 Primary Group을 기준으로 지정되는 경우가 많다. 단, 디렉터리의 setgid 설정 등 다른 규칙이 있으면 달라질 수 있다.

### Supplementary Group

Primary Group 외에 추가로 속하는 그룹이다.

예를 들어 `testuser`를 `ops` 그룹에 추가한다.

```bash
sudo usermod -aG ops testuser
```

옵션 의미:

| 옵션 | 의미 |
|---|---|
| `-G` | Supplementary Groups 목록 지정 |
| `-a` | append. 기존 그룹을 유지하면서 추가 |

> `-G`를 사용할 때 `-a`를 빼면 기존 supplementary group 구성이 바뀔 수 있으므로, 그룹을 **추가**하려는 목적이라면 `-aG`를 함께 사용하는 것이 중요하다.

확인:

```bash
groups testuser
```

또는:

```bash
id testuser
```

---

## 5. `getent`

### 한 줄 정의

`getent`는 **get entries**의 의미로, 시스템이 사용하는 데이터베이스에서 사용자·그룹·호스트 등의 정보를 조회하는 명령어다.

### 그룹 조회

```bash
getent group testuser
```

예시:

```text
testuser:x:1001:
```

필드 의미:

```text
testuser → 그룹 이름
x        → 그룹 비밀번호 필드 자리
1001     → GID
마지막    → 명시적으로 등록된 그룹 멤버 목록
```

### 사용자 조회

```bash
getent passwd testuser
```

시스템의 사용자 데이터베이스를 통해 `testuser` 정보를 조회한다.

### 왜 `/etc/group`를 `grep`하는 것과 다른가?

```bash
grep '^testuser:' /etc/group
```

은 `/etc/group` 파일 자체를 직접 검색한다.

반면:

```bash
getent group testuser
```

는 시스템의 NSS(Name Service Switch) 조회 경로를 통해 그룹 정보를 확인한다.

따라서 로컬 파일뿐 아니라 LDAP 등 외부 사용자/그룹 정보원을 사용하는 시스템에서도 더 범용적으로 사용할 수 있다.

### `id`, `groups`, `getent` 비교

| 명령어 | 중심 질문 |
|---|---|
| `id testuser` | 이 사용자의 UID/GID와 실제 소속 그룹은 무엇인가? |
| `groups testuser` | 이 사용자는 어떤 그룹에 속해 있는가? |
| `getent group ops` | `ops`라는 그룹 자체의 정보는 무엇인가? |
| `getent passwd testuser` | `testuser` 계정 정보는 무엇인가? |

---

## 6. 소유권과 권한을 함께 읽는 방법

다음 파일을 보자.

```text
-rw-r----- 1 linuxuser testuser 0 Sep 3 06:40 secret.conf
```

순서대로 해석한다.

```text
Owner  = linuxuser
Group  = testuser
권한   = 640
```

따라서:

```text
linuxuser → 읽기 + 쓰기
testuser 그룹에 해당하는 사용자 → 읽기
그 외 사용자 → 접근 불가
```

여기서 `testuser`라는 사용자가 실제로 `testuser` 그룹에 속하는지 확인하려면 다음을 사용한다.

```bash
id testuser
```

또는:

```bash
groups testuser
```

---

## 7. 🧪 실습

### 실습 1 - `chown`과 `chgrp` 의미 확인

다음 명령의 의미를 직접 해석했다.

```bash
sudo chown testuser file.txt
```

결과:

```text
파일의 Owner를 testuser로 변경
```

```bash
sudo chown testuser:ops file.txt
```

결과:

```text
Owner → testuser
Group → ops
```

```bash
sudo chgrp ops file.txt
```

결과:

```text
Owner → 기존 값 유지
Group → ops
```

### 실습 2 - `secret.conf` 그룹과 권한 변경

초기 상태:

```bash
ls -ld secret.conf
```

실제 확인 결과:

```text
-rw------- 1 linuxuser linuxuser 0 Sep 3 06:40 secret.conf
```

즉:

```text
Owner  = linuxuser
Group  = linuxuser
권한   = 600
```

파일 Group을 `testuser`로 변경했다.

```bash
sudo chgrp testuser secret.conf
```

확인:

```bash
ls -ld secret.conf
```

결과:

```text
-rw------- 1 linuxuser testuser 0 Sep 3 06:40 secret.conf
```

이 시점에는 Group 이름은 바뀌었지만 권한이 여전히 `600`이므로 Group에는 읽기 권한이 없다.

Group에 읽기 권한을 부여하기 위해 다음을 실행했다.

```bash
chmod 640 secret.conf
```

다시 확인:

```bash
ls -ld secret.conf
```

결과:

```text
-rw-r----- 1 linuxuser testuser 0 Sep 3 06:40 secret.conf
```

최종 상태:

```text
Owner  = linuxuser → rw-
Group  = testuser  → r--
Others             → ---
```

### 실습 결과

**그룹을 바꾸는 것과 그룹 권한을 주는 것은 서로 다른 작업**이라는 점을 확인했다.

```text
chgrp → 어떤 Group이 파일에 연결되는지 변경
chmod → 그 Group이 어떤 rwx 권한을 가지는지 변경
```

---

## 8. ⚠️ 헷갈리기 쉬운 부분

### 사용자 `testuser`와 그룹 `testuser`는 같은 것인가?

아니다.

```text
User testuser  ≠  Group testuser
```

이름이 같을 뿐 서로 다른 사용자/그룹 객체다.

Ubuntu에서는 사용자 생성 시 같은 이름의 Primary Group을 같이 만드는 경우가 일반적이어서 둘이 같은 것처럼 보일 수 있다.

### `chgrp testuser secret.conf`만 실행하면 `testuser`가 바로 읽을 수 있는가?

항상 그런 것은 아니다.

그룹을 `testuser`로 변경해도 Group 권한이 `---`라면 읽을 수 없다.

예:

```text
-rw------- linuxuser testuser secret.conf
```

Group이 `testuser`여도 Group 권한이 없기 때문에 접근할 수 없다.

따라서 필요하다면:

```bash
chmod 640 secret.conf
```

처럼 Group에 읽기 권한도 부여해야 한다.

### `getent group testuser`의 마지막 멤버 필드가 비어 있으면 testuser가 그룹에 속하지 않는가?

그렇게 단정하면 안 된다.

사용자의 **Primary Group**은 `/etc/passwd`의 GID 필드로 연결되며, `/etc/group`의 멤버 목록에는 별도로 표시되지 않을 수 있다.

따라서 특정 사용자의 전체 그룹 소속을 확인할 때는 다음 명령이 더 명확하다.

```bash
id testuser
```

---

## 9. 🔧 Troubleshooting

### 증상

`secret.conf` 파일을 `testuser`가 읽을 수 있도록 만들고 싶었다.

초기 상태:

```text
-rw------- 1 linuxuser linuxuser secret.conf
```

### 원인 분석

현재 권한은 `600`이므로:

```text
Owner  → rw-
Group  → ---
Others → ---
```

`linuxuser`만 읽고 쓸 수 있다.

### 확인 방법

```bash
ls -ld secret.conf
id testuser
```

파일의 Owner/Group과 `testuser`의 그룹 소속을 함께 확인한다.

### 조치

파일 Group을 `testuser`로 변경했다.

```bash
sudo chgrp testuser secret.conf
```

그다음 Group에 읽기 권한을 부여했다.

```bash
chmod 640 secret.conf
```

### 검증

```bash
ls -ld secret.conf
```

결과:

```text
-rw-r----- 1 linuxuser testuser secret.conf
```

필요하면 실제로 `testuser` 세션으로 전환해서 파일 읽기도 검증할 수 있다.

```bash
su - testuser
cat /home/linuxuser/linux-test/secret.conf
```

단, 파일이 위치한 상위 디렉터리들의 `x` 권한도 있어야 경로 접근이 가능하다.

### 배운 점

권한 장애는 `chmod` 숫자 하나만 확인해서는 안 된다.

```text
사용자 그룹 소속
+
파일 Owner/Group
+
파일 rwx
+
상위 디렉터리 접근 권한
```

을 함께 확인해야 한다.

---

## 10. 💼 실무 포인트

운영 환경에서는 여러 사용자에게 파일을 직접 소유시키기보다 **그룹 기반으로 접근 권한을 관리**하는 방식이 자주 사용된다.

예를 들어 애플리케이션 운영 그룹이 `app`이라면:

```bash
sudo chgrp app config.conf
chmod 640 config.conf
```

처럼 설정하고 필요한 사용자들을 `app` 그룹에 추가하는 방식으로 관리할 수 있다.

권한 문제가 발생하면 다음 흐름으로 점검한다.

```text
증상 확인
→ whoami / id
→ ls -l 또는 ls -ld
→ Owner / Group 확인
→ 사용자 그룹 소속 확인
→ 필요한 최소 권한 결정
→ chown/chgrp/chmod 적용
→ 실제 사용자 권한으로 재검증
```

`chmod 777`처럼 문제를 우회하는 방식보다 **정확한 사용자·그룹·권한 구조를 찾아 수정하는 것**이 중요하다.

---

## 11. ✅ 핵심 정리

- 파일에는 **Owner와 Group**이 존재한다.
- `chown user file` → Owner 변경
- `chown user:group file` → Owner와 Group 동시 변경
- `chgrp group file` → Group만 변경
- `chmod`는 소유권이 아니라 `rwx` 권한을 변경한다.
- 사용자는 하나의 **Primary Group**과 여러 **Supplementary Group**에 속할 수 있다.
- Ubuntu에서는 사용자와 같은 이름의 그룹이 함께 생성되는 경우가 일반적이지만 User와 Group은 별개의 객체다.
- `id user`는 UID/GID와 전체 그룹 소속을 확인할 때 유용하다.
- `getent group groupname`은 시스템 그룹 데이터베이스에서 그룹 정보를 조회한다.
- 그룹 소유권을 바꾸는 것만으로는 부족할 수 있으며, `chmod`로 Group 권한도 함께 확인해야 한다.
- 권한 문제는 Owner/Group/`rwx`와 상위 디렉터리 권한까지 함께 분석한다.

---

## 12. 🧠 복습 문제

1. `chown testuser file.txt`와 `chgrp testuser file.txt`의 차이는 무엇인가?
2. `chown testuser:ops file.txt`를 실행하면 어떤 두 정보가 변경되는가?
3. Primary Group과 Supplementary Group의 차이는 무엇인가?
4. `usermod -aG ops testuser`에서 `-a`를 함께 사용하는 이유는 무엇인가?
5. `getent group ops`와 `id testuser`는 각각 무엇을 확인할 때 사용하는가?
6. 파일 Group이 `testuser`인데도 `testuser`가 파일을 읽지 못할 수 있는 이유는 무엇인가?
7. `-rw-r----- linuxuser testuser secret.conf`를 Owner/Group/권한 관점에서 설명해보자.
