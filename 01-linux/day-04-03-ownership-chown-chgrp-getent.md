# Day 4-3 — 파일 소유권, 그룹과 `getent`

> Linux 파일의 Owner/Group이 Permission과 어떻게 연결되는지 이해하고, `chown`, `chgrp`, `id`, `groups`, `getent`를 이용해 실제 접근 주체를 추적했다. 핵심은 **Permission과 Ownership은 서로 다른 정보지만 실제 접근 제어에서는 함께 해석해야 한다**는 점이다.

## 📌 이번에 배운 내용

- File Ownership과 UID/GID
- Owner / Group / Permission 관계
- `chown`, `chgrp`
- Primary / Supplementary Group
- User Private Group
- 새 파일 Group 결정과 SetGID Directory 예고
- NSS와 `getent`
- `id`, `groups`, `getent` 차이
- 실제 `secret.conf` 권한/그룹 변경 실습

## 📚 목차

1. Ownership이란
2. UID/GID와 파일 Metadata
3. `chown`
4. `chgrp`
5. Primary/Supplementary Group
6. 새 파일의 Group
7. NSS와 `getent`
8. 명령어 비교
9. 실제 실습
10. 헷갈리기 쉬운 부분
11. Troubleshooting
12. 실무 포인트
13. 핵심 정리
14. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 의미 |
|---|---|
| `ls -l file` | Permission + Owner + Group 확인 |
| `chown user file` | Owner 변경 |
| `chown user:group file` | Owner와 Group 동시 변경 |
| `chgrp group file` | Group만 변경 |
| `id user` | UID/GID/소속 Group 확인 |
| `groups user` | 사용자 Group 목록 확인 |
| `getent passwd user` | NSS를 통해 사용자 조회 |
| `getent group group` | NSS를 통해 Group 조회 |
| `usermod -aG group user` | Supplementary Group 추가 |

---

## 1. Ownership이란

Linux 파일/Directory에는 Owner UID와 Group GID가 저장된다.

예:

```text
-rw-r----- 1 linuxuser ops 1024 Sep 7 secret.conf
```

여기서:

```text
Owner = linuxuser
Group = ops
Permission = rw-r-----
```

Permission을 해석하려면 Ownership을 함께 알아야 한다.

```text
Owner bits  = rw-
Group bits  = r--
Others bits = ---
```

하지만 Group이 `ops`라는 사실을 모르면 **누가 Group read를 적용받는지** 판단할 수 없다.

---

## 2. UID/GID와 파일 Metadata

파일에는 Username/Group name 자체보다 UID/GID 숫자가 핵심 metadata로 저장된다.

```text
Owner UID = 1000
Group GID = 1002
```

`ls -l`은 시스템의 사용자/그룹 DB를 조회해 사람이 읽기 쉬운 이름으로 변환해서 보여준다.

```text
UID 1000 → linuxuser
GID 1002 → ops
```

그래서 계정이나 Group이 없어지면 이름 대신 숫자로 보일 수 있다.

---

## 3. `chown`

`chown`은 **change owner**다.

### Owner만 변경

```bash
sudo chown testuser file.txt
```

### Owner + Group 변경

```bash
sudo chown testuser:ops file.txt
```

```text
Owner → testuser
Group → ops
```

### 자주 보게 될 옵션

- `-R`: recursive. Directory 아래 전체에 재귀 적용

```bash
sudo chown -R appuser:appgroup /srv/app
```

> `-R`은 영향 범위가 매우 크므로 운영에서는 대상 경로를 반드시 확인한다.

### 왜 일반 사용자는 아무 파일이나 `chown`할 수 없나?

Ownership을 자유롭게 넘길 수 있으면 quota, permission, security model을 우회할 수 있기 때문에 일반적으로 privilege가 필요하다.

---

## 4. `chgrp`

`chgrp`는 **change group**이다.

```bash
sudo chgrp ops file.txt
```

Owner는 유지하고 Group ownership만 바꾼다.

비교:

```text
chmod → Permission bits
chown → Owner 또는 Owner+Group
chgrp → Group ownership
```

---

## 5. Primary / Supplementary Group

사용자는 여러 Group에 속할 수 있다.

### Primary Group

사용자의 기본 Group이다.

```bash
id testuser
```

예:

```text
uid=1001(testuser) gid=1001(testuser) groups=1001(testuser),1002(ops)
```

`gid=1001(testuser)`가 Primary Group이다.

### Supplementary Group

추가 Group membership이다.

```text
ops
sudo
users
```

등 역할별 접근에 활용할 수 있다.

### User Private Group

Ubuntu/Debian에서는 사용자 생성 시 같은 이름의 Group을 함께 만드는 방식이 흔하다.

```text
User  testuser
Group testuser
```

이름은 같아도 서로 다른 객체다.

---

## 6. 새 파일의 Group은 어떻게 정해지나?

일반적으로 사용자가 새 파일을 만들면 자신의 effective GID/Primary Group이 기본 Group ownership에 영향을 준다.

예:

```text
User = testuser
Primary Group = testuser
```

보통:

```bash
touch new.txt
```

→ Group `testuser`.

하지만 부모 Directory에 **SetGID**가 설정되어 있으면 부모 Directory Group을 상속할 수 있다.

즉:

```text
일반 Directory → 생성자의 Group 영향
SetGID Directory → 부모 Directory Group 상속
```

이 내용은 Day 4-4에서 실습한다.

---

## 7. NSS와 `getent`

### `getent`

`getent`는 **get entries**다. NSS가 사용하는 database에서 entry를 조회한다.

```bash
getent passwd testuser
getent group ops
```

### 왜 `grep /etc/group`만 보면 부족할 수 있나?

기업 Linux 서버에서는 LDAP, SSSD 등 외부 Identity Source를 사용할 수 있다.

```text
grep /etc/group → 로컬 파일만
getent group    → 시스템의 NSS 조회 경로
```

따라서 실제 시스템이 인식하는 계정/그룹 확인에는 `getent`가 유용하다.

### `getent group`의 member list 주의

예:

```text
testuser:x:1001:
```

마지막이 비어 있어도 `testuser` 사용자의 Primary GID가 1001이라면 그 사용자는 해당 Group의 Primary member일 수 있다.

즉 `getent group` 마지막 필드만 보고 전체 membership을 판단하지 않는다.

---

## 8. `id`, `groups`, `getent` 비교

| 명령어 | 질문 |
|---|---|
| `id testuser` | 이 사용자의 UID/GID와 실제 Group credential은? |
| `groups testuser` | 이 사용자가 속한 Group 이름은? |
| `getent passwd testuser` | 이 사용자 DB entry는? |
| `getent group ops` | 이 Group DB entry는? |

권한 문제에서는 보통 `id`가 가장 직접적인 출발점이다.

---

## 9. 🧪 실제 실습 — `secret.conf`

초기:

```text
-rw------- 1 linuxuser linuxuser ... secret.conf
```

즉:

```text
Owner = linuxuser
Group = linuxuser
Mode  = 600
```

Group 변경:

```bash
sudo chgrp testuser secret.conf
```

결과:

```text
-rw------- 1 linuxuser testuser ... secret.conf
```

하지만 아직 Group Permission은 `---`다.

따라서:

```bash
chmod 640 secret.conf
```

결과:

```text
-rw-r----- 1 linuxuser testuser ... secret.conf
```

이제:

```text
Owner linuxuser → rw-
Group testuser  → r--
Others          → ---
```

### 중요한 결론

```text
Group을 바꿈 ≠ Group에게 권한을 줌
```

두 작업은 별개다.

```text
chgrp → 어떤 Group인가
chmod → 그 Group에 어떤 Permission인가
```

---

## 10. ⚠️ 헷갈리기 쉬운 부분

> ⚠️ User `testuser`와 Group `testuser`는 이름만 같을 수 있는 별개 객체다.

> ⚠️ `chgrp testuser secret.conf`만으로 `testuser` 사용자가 파일을 읽을 수 있게 되는 것은 아니다. Group permission도 필요하다.

> ⚠️ 파일이 `640`이어도 부모 Directory에 traverse `x`가 없으면 접근이 실패할 수 있다.

> ⚠️ `getent group` 마지막 member list는 Primary membership 전체를 보여주는 목록이 아니다.

---

## 11. 🔧 Troubleshooting

### 증상

`secret.conf` Group을 `testuser`로 바꿨는데 testuser가 읽지 못한다.

### 확인

```bash
id testuser
ls -l secret.conf
ls -ld .
```

### 가설

- testuser가 Group에 속하지 않는가?
- Group bit에 `r`이 없는가?
- 부모 Directory `x`가 없는가?

### 원인 예

Mode가 여전히 `600`.

### 조치

요구사항이 Group read라면:

```bash
chmod 640 secret.conf
```

### 검증

실제로 `testuser`로 전환해 읽기 테스트한다.

---

## 12. 💼 실무 포인트

Permission 장애는 다음 네 축을 같이 본다.

```text
Who?       → id
Ownership? → ls -l
Mode?      → ls -l / stat
Path?      → ls -ld parent dirs
```

운영에서는 `chmod 777`보다 먼저 **Identity와 Ownership mismatch**를 의심해야 하는 경우가 많다.

공유 Directory는 Owner/Group, Supplementary Group, SetGID, umask를 함께 설계하는 것이 중요하다.

---

## 13. ✅ 핵심 정리

- 파일 Ownership은 UID/GID 기반이다.
- `chmod`와 `chown/chgrp`는 역할이 다르다.
- Primary Group은 기본 Group, Supplementary Group은 추가 membership이다.
- 새 파일 Group은 생성자 Group이나 SetGID parent에 의해 결정될 수 있다.
- `getent`는 NSS 경로를 통해 계정/그룹 DB를 조회한다.
- 실제 접근 여부는 Identity + Ownership + Permission + Path를 함께 봐야 한다.

---

## 14. 🧠 복습 문제

1. Ownership과 Permission의 차이는?
2. 파일에 Username 대신 UID가 저장되는 이유는?
3. `chown user:group file`을 해석해보라.
4. `chgrp`와 `chmod`의 차이는?
5. Primary/Supplementary Group은 어떻게 다른가?
6. `getent`가 실무에서 유용한 이유는?
7. Group을 바꿨는데도 읽기 실패가 날 수 있는 이유 3가지를 말해보라.
