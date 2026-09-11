# Day 4-4 — Linux 특수 권한과 `sudo`

> 일반 `rwx`를 넘어 **SetUID, SetGID, Sticky Bit**이 어떤 문제를 해결하기 위해 존재하는지 학습했다. 숫자 `4/2/1` 특수 비트와 `s/t` 표시를 단순 암기하지 않고 Effective UID/GID, 공유 Directory, 삭제 권한과 연결해서 이해한다.

## 📌 이번에 배운 내용

- 특수 Permission bit 개념
- SetUID = 4
- SetGID = 2
- Sticky Bit = 1
- `s`, `S`, `t`, `T`
- Real UID / Effective UID 기초
- `/usr/bin/passwd`와 SetUID
- SetGID Directory Group inheritance
- Sticky Bit와 `/tmp`
- `sudo`와 SetUID의 차이
- 실제 `/tmp/team-share` 공유 Directory 실습

## 📚 목차

1. 특수 Permission이 필요한 이유
2. 숫자 표현 구조
3. Real/Effective UID
4. SetUID
5. SetGID
6. Sticky Bit
7. `s/S/t/T`
8. `sudo`
9. 실제 실습
10. 헷갈리기 쉬운 부분
11. Troubleshooting
12. 실무 포인트
13. 핵심 정리
14. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 의미 |
|---|---|
| `ls -l /usr/bin/passwd` | SetUID 대표 실행 파일 확인 |
| `chmod u+s file` | SetUID 설정 |
| `chmod g+s dir` | SetGID 설정 |
| `chmod +t dir` | Sticky Bit 설정 |
| `chmod 4755 file` | SetUID + `755` |
| `chmod 2770 dir` | SetGID + `770` |
| `chmod 1777 dir` | Sticky + `777` |
| `stat -c '%a %A' path` | 숫자/문자 Permission 함께 확인 |
| `sudo -l` | 현재 사용자의 sudo 허용 범위 확인 |

---

## 1. 특수 Permission이 필요한 이유

기본 `rwx`만으로 해결하기 어려운 세 가지 대표 문제가 있다.

```text
1. 특정 실행 파일이 제한된 높은 권한을 필요로 함
2. 공유 Directory에서 새 파일의 Group을 통일하고 싶음
3. 모두가 쓰는 Directory에서 남의 파일 삭제는 막고 싶음
```

이를 위해 SetUID, SetGID, Sticky Bit가 있다.

---

## 2. 숫자 표현 구조

기본 mode `755` 앞에 특수 bit 한 자리를 추가할 수 있다.

```text
특수 | Owner | Group | Others
  4      7       5       5
```

특수 bit도 bit 조합이다.

```text
SetUID    = 4
SetGID    = 2
Sticky    = 1
```

그래서:

```text
4755 → SetUID + rwxr-xr-x
2750 → SetGID + rwxr-x---
1777 → Sticky + rwxrwxrwx
6755 → SetUID(4)+SetGID(2)+755
```

일반 `r=4,w=2,x=1`과 숫자가 같아 보이지만 **위치가 다르다.** 앞자리 특수 bit와 뒤의 세 자리 기본 Permission을 구분한다.

---

## 3. Real UID와 Effective UID

### Real UID

프로세스를 실제로 실행한 사용자의 Identity를 나타내는 개념이다.

### Effective UID

Kernel이 많은 Permission check에서 실제로 사용하는 실행 권한 Identity다.

일반 실행에서는 보통:

```text
Real UID = linuxuser
Effective UID = linuxuser
```

SetUID root 실행 파일을 실행하면 프로그램 실행 동안:

```text
Real UID      = 일반 사용자
Effective UID = 파일 Owner(root)
```

처럼 달라질 수 있다.

이 덕분에 일반 사용자가 특정 프로그램을 통해 제한된 privileged operation을 수행할 수 있다.

---

## 4. SetUID

### 한 줄 정의

실행 파일에 SetUID가 있으면 실행 중 Effective UID가 파일 Owner 기준으로 동작할 수 있다.

대표 예:

```bash
ls -l /usr/bin/passwd
```

예:

```text
-rwsr-xr-x 1 root root ... /usr/bin/passwd
```

Owner execute 위치의 `s`가 SetUID를 나타낸다.

### 왜 `passwd`에 필요한가?

일반 사용자는 `/etc/shadow`를 직접 수정할 수 없어야 한다. 하지만 자신의 Password는 변경할 수 있어야 한다.

```text
일반 사용자
→ /usr/bin/passwd 실행
→ 필요한 순간 privileged access
→ 프로그램 내부 정책 검사
→ 허용된 password 변경
```

중요:

```text
SetUID 있음
≠
사용자가 root의 모든 권한을 자유롭게 사용
```

실행 파일 내부의 로직과 검증이 여전히 중요하다.

### 보안 관점

SetUID root 프로그램의 취약점은 privilege escalation로 이어질 수 있어서 운영에서 매우 민감한 대상이다.

---

## 5. SetGID

### 실행 파일

실행 시 Effective GID가 파일 Group 기준으로 동작할 수 있다.

### Directory

운영에서 더 자주 체감하는 사용법이다.

```bash
chmod g+s /srv/team-share
```

SetGID Directory 아래 새 파일/Directory는 **부모 Directory의 Group을 상속**한다.

예:

```text
/srv/team-share
Group = ops
SetGID = ON
```

`testuser`가 내부에 파일을 만들더라도:

```text
new.txt Group → ops
```

처럼 유지할 수 있다.

이것이 협업 Directory에서 유용한 이유다.

---

## 6. Sticky Bit

### 한 줄 정의

공용 writable Directory에서 파일 삭제/rename을 제한하여 **아무 사용자나 다른 사용자의 Entry를 지우지 못하게** 하는 특수 bit다.

대표:

```bash
ls -ld /tmp
```

보통:

```text
drwxrwxrwt
```

숫자:

```text
1777
```

`/tmp`는 모두가 파일을 만들 수 있어야 하지만, Sticky Bit가 없으면 Directory write permission 때문에 다른 사용자의 파일을 삭제할 수 있는 문제가 생긴다.

Sticky Bit가 있으면 일반적으로 파일 Owner, Directory Owner, privileged user가 삭제/rename 권한을 갖도록 제한한다.

---

## 7. `s`, `S`, `t`, `T`

### 소문자 `s`

특수 bit + execute bit 둘 다 있음.

```text
rws
r-s
```

### 대문자 `S`

SetUID/SetGID bit는 있지만 해당 execute bit가 없음.

```text
rwS
```

즉 설정은 되어 있지만 실제 실행 의미가 이상하거나 불완전할 수 있어 점검 포인트가 된다.

### 소문자 `t`

Sticky Bit + Others execute 있음.

### 대문자 `T`

Sticky Bit는 있지만 Others execute가 없음.

이 대소문자 차이는 `ls -l`만 보고 특수 bit와 execute bit를 동시에 해석할 수 있게 한다.

---

## 8. `sudo`

### `sudo`와 SetUID 차이

둘 다 privilege와 관련 있지만 목적이 다르다.

```text
SetUID → 파일 Permission bit 기반 실행 권한 전환
sudo   → sudoers 정책 기반 명령 실행 권한 위임
```

`sudo`는 누가 어떤 명령을 어떤 사용자 권한으로 실행할 수 있는지 정책으로 통제하고 audit/logging과 결합하기 쉽다.

### `sudo -l`

```bash
sudo -l
```

`-l` = list. 현재 사용자에게 허용된 sudo command를 확인한다.

### sudo를 언제 붙이나?

무조건 붙이는 습관은 좋지 않다.

```text
현재 UID/GID로 충분한가?
→ 충분하면 일반 권한 사용
→ 부족하고 정책상 허용되면 sudo
```

최소 권한 원칙과 연결된다.

---

## 9. 🧪 실제 실습 — `/tmp/team-share`

목표:

```text
Owner = linuxuser
Group = testuser
Mode  = rwxrws---
```

생성:

```bash
mkdir /tmp/team-share
sudo chgrp testuser /tmp/team-share
chmod 770 /tmp/team-share
chmod g+s /tmp/team-share
```

검증:

```bash
ls -ld /tmp/team-share
```

예:

```text
drwxrws--- linuxuser testuser /tmp/team-share
```

다른 사용자로:

```bash
su - testuser
```

내부 항목 생성 후 Group inheritance를 확인했다.

### 실습 중 실수 1

```bash
mkdir /tmp/team-share/testfile.txt
```

`.txt` 이름 때문에 파일처럼 보이지만 `mkdir`는 Directory를 만든다.

파일은:

```bash
touch /tmp/team-share/testfile.txt
```

### 실습 중 실수 2

```bash
sudo chown testuser /tmp/team-share
```

은 Owner까지 `testuser`로 바꾸므로 요구사항과 달라진다.

Group만 바꾸려면:

```bash
chgrp testuser /tmp/team-share
```

---

## 10. ⚠️ 헷갈리기 쉬운 부분

> ⚠️ `4`, `2`, `1`은 여기서 특수 bit 앞자리 값이다. 기본 `r=4,w=2,x=1`과 위치/의미를 구분한다.

> ⚠️ SetUID는 Directory에 일반적으로 의미 있게 사용하지 않는다. 핵심은 executable file이다.

> ⚠️ SetGID는 executable에도 의미가 있지만 운영에서는 shared Directory Group inheritance로 매우 중요하다.

> ⚠️ Sticky Bit는 file보다는 Directory에서 실무적으로 중요하다.

> ⚠️ `chmod +t`에서 `t`는 Others의 일반 permission이 아니라 특수 bit다. 표시만 Others execute 위치에 나타난다.

---

## 11. 🔧 Troubleshooting

### 증상

공유 Directory 안 새 파일 Group이 제각각이다.

### 확인

```bash
ls -ld /srv/share
ls -l /srv/share
id user
```

### 가설

부모 Directory에 SetGID가 없다.

### 조치

필요한 Group ownership을 먼저 맞춘 뒤:

```bash
sudo chgrp ops /srv/share
sudo chmod g+s /srv/share
```

필요한 기본 Permission도 설정한다.

### 검증

서로 다른 사용자로 새 파일을 만든 뒤 Group이 `ops`로 상속되는지 확인한다.

---

## 12. 💼 실무 포인트

공유 Directory 설계는 다음을 함께 본다.

```text
Owner / Group
기본 rwx
SetGID
Sticky Bit 필요 여부
umask
사용자 Group membership
```

SetUID root 파일은 보안 점검 대상이다. 필요 없는 SetUID binary가 있는지 확인하는 audit 작업도 있다.

`sudo`는 일반적으로 SetUID 기반 무제한 privilege보다 **정책 기반 최소 권한 위임**에 더 적합한 운영 수단이다.

---

## 13. ✅ 핵심 정리

- 특수 bit는 SetUID=4, SetGID=2, Sticky=1이다.
- SetUID executable은 Effective UID를 파일 Owner 기준으로 사용할 수 있다.
- SetGID Directory는 새 항목의 Group inheritance를 통일한다.
- Sticky Bit는 공용 writable Directory의 삭제/rename을 보호한다.
- `s/S/t/T`는 특수 bit와 execute bit의 조합 상태를 보여준다.
- `sudo`는 sudoers 정책 기반 privilege delegation이다.

---

## 14. 🧠 복습 문제

1. `4755`, `2750`, `1777`을 각각 해석해보라.
2. Real UID와 Effective UID의 차이는?
3. `/usr/bin/passwd`가 SetUID를 사용하는 이유는?
4. SetGID Directory가 협업에 유용한 이유는?
5. `/tmp`에 Sticky Bit가 필요한 이유는?
6. `s`와 `S`, `t`와 `T`의 차이는?
7. `sudo`와 SetUID는 어떤 점이 다른가?
