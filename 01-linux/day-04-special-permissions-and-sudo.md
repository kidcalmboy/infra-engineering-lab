# Day 4-4 - Linux 특수 권한과 sudo 이해

> SetUID, SetGID, Sticky Bit의 동작 원리를 이해하고, 실제 공유 디렉터리 실습을 통해 특수 권한과 `sudo` 사용 기준을 확인했다.

## 📌 이번에 배운 내용

이번 학습에서는 일반 권한 `rwx`에 더해 Linux의 특수 권한을 학습했다.

주요 학습 내용:

- SetUID: 실행 파일이 파일 Owner의 권한으로 동작하는 원리
- `/usr/bin/passwd`와 SetUID의 관계
- SetGID: 공유 디렉터리에서 Group 상속
- Sticky Bit: 공유 디렉터리에서 다른 사용자의 파일 삭제 제한
- `/usr/bin`과 `usr/bin`의 절대/상대 경로 차이
- `sudo`가 필요한 경우와 필요하지 않은 경우
- 직접 `/tmp/team-share`를 구성하고 Group 상속 검증
- `mkdir`와 `touch` 차이로 인한 실습 오류 확인

## 📚 목차

1. 특수 권한 개요
2. SetUID
3. SetGID
4. Sticky Bit
5. 절대 경로와 상대 경로
6. `sudo` 사용 기준
7. 실습
8. 헷갈리기 쉬운 부분
9. Troubleshooting
10. 실무 포인트
11. 핵심 정리
12. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 의미 | 언제 사용하는가 |
|---|---|---|
| `ls -l /usr/bin/passwd` | `passwd` 실행 파일의 권한 확인 | SetUID 확인 |
| `chmod u+s file` | SetUID 추가 | 실행 파일에 Owner 권한 기반 실행이 필요할 때 |
| `chmod g+s directory` | SetGID 추가 | 공유 디렉터리의 Group 상속 |
| `chmod +t directory` | Sticky Bit 추가 | 공용 쓰기 디렉터리에서 삭제 제한 |
| `chmod 2750 directory` | SetGID + 일반 권한 설정 | 숫자 방식으로 특수 권한 설정 |
| `chmod 1777 directory` | Sticky Bit + `777` | `/tmp`와 같은 공용 임시 디렉터리 구조 |
| `sudo -l` | 현재 사용자의 sudo 허용 범위 확인 | sudo 정책 확인 |
| `whoami` | 현재 사용자 확인 | root인지 일반 사용자인지 확인 |
| `id` | UID/GID/그룹 확인 | 사용자 권한 분석 |

---

## 1. 특수 권한 개요

### 한 줄 정의

특수 권한은 일반적인 `rwx`만으로 표현하기 어려운 **실행 권한 위임, 그룹 상속, 공유 디렉터리 보호**를 위해 사용하는 추가 권한 비트다.

| 특수 권한 | 숫자 | 대표 사용 대상 | 핵심 의미 |
|---|---:|---|---|
| SetUID | `4` | 실행 파일 | 실행 시 파일 Owner 권한 사용 |
| SetGID | `2` | 실행 파일/디렉터리 | 실행 시 Group 권한 사용 / 디렉터리에서는 Group 상속 |
| Sticky Bit | `1` | 디렉터리 | 다른 사용자가 만든 파일의 삭제/이름 변경 제한 |

일반 권한 앞에 한 자리를 추가해 숫자로 표현할 수 있다.

```text
755   → 일반 권한
4755  → SetUID + 755
2755  → SetGID + 755
1777  → Sticky Bit + 777
```

---

## 2. SetUID

### 한 줄 정의

SetUID는 **실행 파일을 실행한 사용자가 아니라 그 파일의 Owner 권한을 기준으로 프로그램이 동작할 수 있게 하는 특수 권한**이다.

### 예시: `/usr/bin/passwd`

```bash
ls -l /usr/bin/passwd
```

대표적으로 다음과 비슷한 형태가 보인다.

```text
-rwsr-xr-x 1 root root ... /usr/bin/passwd
```

Owner 권한의 실행 위치가 `x`가 아니라 `s`다.

```text
rws
  ↑
SetUID
```

`passwd`의 Owner는 `root`다. 일반 사용자는 `/etc/shadow`를 직접 수정할 권한이 없지만, 자신의 비밀번호를 바꿀 수 있어야 한다.

개념 흐름:

```text
일반 사용자
→ passwd 실행
→ passwd는 root 소유 + SetUID
→ 필요한 범위에서 높은 권한으로 인증 정보 변경
```

### 중요한 구분

`/usr/bin/passwd`를 일반 사용자가 실행할 수 있다고 해서 **모든 사용자의 비밀번호를 마음대로 변경할 수 있다는 뜻은 아니다.**

프로그램 내부에서 현재 사용자, 대상 계정, 인증 상태 등을 검사한다.

```text
실행 권한이 있음
≠
프로그램의 모든 기능을 사용할 수 있음
```

---

## 3. SetGID

### 한 줄 정의

디렉터리에 SetGID를 설정하면 **그 안에서 새로 생성되는 파일과 디렉터리가 부모 디렉터리의 Group을 상속받는다.**

예:

```text
/tmp/team-share
Owner = linuxuser
Group = testuser
```

SetGID를 적용하면 내부에서 새로 생성되는 항목은 생성자의 Primary Group이 아니라 부모 디렉터리의 Group인 `testuser`를 상속할 수 있다.

```bash
chmod g+s /tmp/team-share
```

또는 숫자 방식:

```bash
chmod 2770 /tmp/team-share
```

확인:

```bash
ls -ld /tmp/team-share
```

예:

```text
drwxrws---
      ↑
    SetGID
```

### 실무에서 중요한 이유

공유 디렉터리에서 여러 사용자가 파일을 만들면 Group이 제각각이면 협업 권한 관리가 어려워진다.

SetGID를 사용하면 **공유 디렉터리의 Group을 일관되게 유지**할 수 있다.

---

## 4. Sticky Bit

### 한 줄 정의

Sticky Bit는 공용 쓰기 디렉터리에서 **다른 사용자가 만든 파일을 임의로 삭제하거나 이름을 변경하지 못하도록 제한**하는 특수 권한이다.

대표적인 예:

```bash
ls -ld /tmp
```

보통 다음과 비슷하다.

```text
drwxrwxrwt
         ↑
         t
```

숫자 권한으로는 보통:

```text
1777
```

이다.

`/tmp`는 여러 사용자가 파일을 만들 수 있어야 하므로 넓은 쓰기 권한이 필요하지만, Sticky Bit가 없으면 다른 사용자의 파일까지 지우는 문제가 생길 수 있다.

### 왜 `chmod +t`인가?

Sticky Bit는 `r`, `w`, `x`처럼 단순한 Others 권한 하나가 아니라 **별도의 특수 권한 비트**다.

```bash
chmod +t directory
```

형태로 설정하는 것이 가장 직관적이다.

일부 시스템에서는 다음 형식도 동작할 수 있다.

```bash
chmod o+t directory
```

하지만 의미상 `t`는 단순한 Others 권한이 아니라 특수 권한이며, 화면에 표시될 때 Others의 `x` 위치에 나타날 뿐이다.

---

## 5. `/usr/bin`과 `usr/bin`

### 한 줄 정의

맨 앞 `/`가 있으면 **절대 경로**, 없으면 **현재 디렉터리 기준 상대 경로**다.

```bash
cd /usr/bin
```

→ 루트(`/`)에서 시작하는 실제 `/usr/bin`으로 이동한다.

```bash
cd usr/bin
```

→ 현재 위치 아래의 `usr/bin`을 찾는다.

예를 들어 현재 위치가 `/home/linuxuser`라면:

```text
usr/bin
→ /home/linuxuser/usr/bin
```

을 찾으므로 해당 경로가 없으면 `No such file or directory`가 발생한다.

---

## 6. `sudo`는 언제 사용하는가?

### 한 줄 정의

`sudo`는 **현재 사용자가 원래 가진 권한을 넘어서는 작업을 관리자 권한으로 실행할 때** 사용한다.

현재 환경의 `linuxuser`는 root가 아니라 일반 사용자이며, sudo 정책에 따라 필요한 순간에 관리자 권한을 사용한다.

```bash
whoami
```

예:

```text
linuxuser
```

현재 sudo 범위는 다음으로 확인할 수 있다.

```bash
sudo -l
```

### 판단 기준

```text
내 권한으로 작업 가능한가?
→ 가능: sudo 불필요
→ 불가능하거나 시스템 변경: sudo 필요 가능성 높음
```

| 상황 | 일반적인 sudo 필요 여부 |
|---|---|
| 내 홈에서 파일 생성 | X |
| 내가 소유한 파일 `chmod` | X |
| `/etc` 설정 수정 | O |
| 사용자 생성/삭제 | O |
| 다른 사용자로 `chown` | O |
| 서비스 재시작 | O |
| `/tmp`에 디렉터리 생성 | 보통 X |

> `sudo`를 습관적으로 붙이지 말고 **현재 사용자와 대상 파일/디렉터리의 권한을 먼저 판단**하는 습관이 중요하다.

---

## 7. 🧪 실습 - `/tmp/team-share`

### 실습 목표

`linuxuser`가 소유하고 `testuser` 그룹이 사용하는 공유 디렉터리를 구성한 뒤, SetGID에 의해 새 항목의 Group이 상속되는지 직접 확인한다.

### 1. 공유 디렉터리 생성

```bash
mkdir /tmp/team-share
```

초기 확인:

```bash
ls -ld /tmp/team-share
```

실제 초기 상태:

```text
drwxrwxr-x ... linuxuser linuxuser ... /tmp/team-share
```

### 2. Group 변경

```bash
sudo chgrp testuser /tmp/team-share
```

### 3. 일반 권한 설정

```bash
sudo chmod 770 /tmp/team-share
```

### 4. SetGID 설정

```bash
sudo chmod g+s /tmp/team-share
```

검증:

```bash
ls -ld /tmp/team-share
```

결과:

```text
drwxrws--- ... linuxuser testuser ... /tmp/team-share
```

즉:

```text
Owner = linuxuser
Group = testuser
Owner  → rwx
Group  → rwx + SetGID
Others → ---
```

### 5. 다른 사용자로 전환

```bash
su - testuser
```

### 6. 내부 항목 생성 및 확인

실습에서는 다음 명령을 사용했다.

```bash
mkdir /tmp/team-share/testfile.txt
```

그리고:

```bash
ls -l /tmp/team-share
```

결과:

```text
drwxrwsr-x ... testuser testuser ... testfile.txt
```

새 항목의 Group이 `testuser`가 되었으므로 **SetGID에 의한 Group 상속은 정상적으로 확인되었다.**

다만 `mkdir`를 사용했기 때문에 `testfile.txt`라는 이름의 **파일이 아니라 디렉터리**가 생성되었다.

파일을 만들려면 다음을 사용해야 한다.

```bash
touch /tmp/team-share/testfile.txt
```

---

## 8. ⚠️ 헷갈리기 쉬운 부분

### `mkdir testfile.txt`를 하면 파일이 만들어지는가?

아니다.

`mkdir`은 이름에 `.txt`가 붙어 있어도 **항상 디렉터리를 생성**한다.

```bash
mkdir testfile.txt
```

→ `testfile.txt`라는 디렉터리

```bash
touch testfile.txt
```

→ `testfile.txt`라는 일반 파일

`ls -l`의 첫 문자를 보면 구분할 수 있다.

```text
d → directory
- → regular file
```

### `sudo chown testuser /tmp/team-share`가 왜 문제였는가?

실습 요구사항은:

```text
Owner = linuxuser
Group = testuser
```

였다.

그런데 다음 명령을 실행하면:

```bash
sudo chown testuser /tmp/team-share
```

Owner까지 `testuser`로 변경된다.

따라서 요구사항과 다른 상태가 된다.

> Group만 바꾸고 싶다면 `chgrp`, Owner를 바꾸려면 `chown`을 사용한다.

### `testuser`에서 `sudo chown`이 거부된 이유

`testuser`는 sudo 권한이 없기 때문에 다음과 같은 메시지가 발생했다.

```text
sudo: I'm sorry testuser. I'm afraid I can't do that
```

즉 모든 일반 사용자가 sudo를 사용할 수 있는 것은 아니다.

---

## 9. 🔧 Troubleshooting

### 증상 1 - 파일을 만들려고 했는데 디렉터리가 생성됨

실행:

```bash
mkdir /tmp/team-share/testfile.txt
```

확인:

```bash
ls -l /tmp/team-share
```

결과의 첫 글자가 `d`였다.

### 원인

`mkdir`은 디렉터리 생성 명령어다. 확장자 `.txt`는 파일 종류를 결정하지 않는다.

### 해결

파일 생성에는 다음을 사용한다.

```bash
touch /tmp/team-share/testfile.txt
```

### 배운 점

Linux에서는 이름이나 확장자보다 **명령어와 실제 파일 타입**이 중요하다.

---

### 증상 2 - 공유 디렉터리 Owner가 요구사항과 달라짐

실행:

```bash
sudo chown testuser /tmp/team-share
```

### 원인

`chown`은 Owner를 변경하는 명령어이므로 기존 `linuxuser` 소유권이 `testuser`로 바뀌었다.

### 해결 방향

Owner/Group을 원하는 상태로 다시 맞춘 뒤 `ls -ld`로 검증해야 한다.

### 배운 점

권한 문제에서는 `chmod`만 볼 것이 아니라 다음 세 가지를 함께 확인해야 한다.

```text
Owner
Group
Permission
```

---

## 10. 💼 실무 포인트

공유 디렉터리를 구성할 때는 단순히 `chmod 777`로 해결하면 안 된다.

권장 사고 흐름:

```text
누가 사용할 것인가?
→ 어떤 Group으로 묶을 것인가?
→ Owner/Group 설정
→ 필요한 rwx만 부여
→ SetGID 등 특수 권한 필요 여부 판단
→ 다른 사용자로 실제 생성 테스트
→ ls -l / ls -ld로 검증
```

또한 관리자 권한이 필요한지 판단하기 위해 **현재 사용자, 대상의 Owner/Group, 현재 권한**을 먼저 확인해야 한다.

```bash
whoami
id
ls -l 대상파일
ls -ld 대상디렉터리
sudo -l
```

---

## 11. ✅ 핵심 정리

- SetUID는 실행 파일을 **파일 Owner 권한 기준으로 실행**할 수 있게 한다.
- `/usr/bin/passwd`는 대표적인 SetUID 사례다.
- 일반 사용자가 `passwd`를 실행할 수 있어도 모든 비밀번호를 마음대로 변경할 수 있는 것은 아니다.
- SetGID를 디렉터리에 설정하면 새 파일/디렉터리가 부모의 Group을 상속한다.
- Sticky Bit는 공용 쓰기 디렉터리에서 다른 사용자의 파일 삭제를 제한한다.
- `/usr/bin`은 절대 경로, `usr/bin`은 현재 위치 기준 상대 경로다.
- `sudo`는 현재 사용자의 권한을 넘어서는 작업에서 필요하다.
- `mkdir`은 디렉터리, `touch`는 일반 파일 생성에 사용한다.
- `chgrp`는 Group 변경, `chown`은 Owner 변경에 사용한다.
- 권한 문제는 **Owner + Group + rwx + 특수 권한**을 함께 확인해야 한다.

---

## 12. 🧠 복습 문제

1. SetUID가 설정된 실행 파일은 어떤 권한을 기준으로 실행되는가?
2. 일반 사용자가 `/usr/bin/passwd`를 실행할 수 있는 이유는 무엇인가?
3. `passwd`를 실행할 수 있다고 해서 다른 사용자의 비밀번호도 자유롭게 바꿀 수 있는가?
4. 디렉터리에 SetGID를 적용하면 새 파일의 Group은 어떻게 결정되는가?
5. `/tmp`에 Sticky Bit가 필요한 이유는 무엇인가?
6. `/usr/bin`과 `usr/bin`의 차이는 무엇인가?
7. `mkdir file.txt`와 `touch file.txt`의 결과 차이는 무엇인가?
8. `sudo`가 필요한지 판단하기 전에 무엇을 확인해야 하는가?
9. `chown`과 `chgrp`의 차이는 무엇인가?
