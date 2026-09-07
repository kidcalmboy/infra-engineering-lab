# Day 4-1 — Linux 파일 권한 기초와 `chmod`

> Linux의 기본 접근 제어 모델인 **Owner / Group / Others + `rwx`** 구조를 학습했다. 숫자 권한 `4/2/1`을 단순 암기하지 않고 **왜 4·2·1인지, `640`이 어떻게 `rw-r-----`가 되는지, Kernel이 누구에게 어느 권한을 적용하는지**까지 이해하는 것을 목표로 한다.

## 📌 이번에 배운 내용

- 파일 Metadata와 Permission
- Owner / Group / Others
- `r` / `w` / `x`
- `r=4`, `w=2`, `x=1`의 의미
- 3-bit와 8진수(octal) Permission
- `600`, `640`, `644`, `700`, `750`, `755`, `777`
- `ls -l`, `ls -ld`
- `chmod` 숫자 방식
- 최소 권한 원칙

## 📚 목차

1. Permission이 필요한 이유
2. Owner / Group / Others
3. `rwx` 의미
4. 왜 `r=4`, `w=2`, `x=1`인가
5. 숫자 권한 계산
6. `ls -l` 출력 해석
7. `chmod`
8. 실제 실습
9. 헷갈리기 쉬운 부분
10. Troubleshooting
11. 실무 포인트
12. 핵심 정리
13. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 의미 |
|---|---|
| `ls -l file` | 파일의 타입·권한·Owner·Group 등 상세 확인 |
| `ls -ld dir` | 디렉터리 내부가 아니라 디렉터리 자체 정보 확인 |
| `chmod 600 file` | Owner만 읽기/쓰기 |
| `chmod 640 file` | Owner `rw`, Group `r`, Others 없음 |
| `chmod 644 file` | Owner `rw`, Group/Others `r` |
| `chmod 755 file` | Owner `rwx`, Group/Others `r-x` |

---

## 1. Permission이 필요한 이유

Linux는 Multi-user OS다. 여러 사용자와 서비스 Process가 같은 시스템의 파일을 공유하므로 누가 읽고, 수정하고, 실행할 수 있는지 통제해야 한다.

파일/디렉터리에는 Ownership과 Permission metadata가 있다.

```text
Owner
Group
Permission bits
```

Kernel은 현재 Process의 credential(UID/GID 등)과 파일 metadata를 비교해 접근 허용 여부를 판단한다.

---

## 2. Owner / Group / Others

기본 Permission은 세 범주로 나뉜다.

```text
Owner  → 파일 소유 사용자
Group  → 파일에 연결된 소유 그룹
Others → 위 두 범주에 해당하지 않는 사용자
```

예:

```text
-rw-r----- 1 linuxuser ops 1024 Sep 7 app.conf
```

```text
Owner = linuxuser
Group = ops
```

권한은 순서대로:

```text
rw- | r-- | ---
Owner Group Others
```

### 실제로 어느 칸을 적용받나?

단순히 `Owner → Group → Others를 모두 합쳐서` 권한을 계산하는 방식이 아니다. 일반적인 전통 Unix permission check에서는 **해당 사용자에게 적용되는 한 class**의 bits를 기준으로 판단한다.

예를 들어 사용자가 Owner라면 Owner bits를 보고, Owner가 아니지만 파일 Group과 맞으면 Group bits를 본다.

---

## 3. `r`, `w`, `x`

파일 기준:

| 기호 | 영어 | 의미 |
|---|---|---|
| `r` | read | 파일 내용 읽기 |
| `w` | write | 파일 내용 수정 |
| `x` | execute | 실행 파일/스크립트를 실행할 수 있는 Permission |

```text
rw- → 읽기 O, 쓰기 O, 실행 X
r-x → 읽기 O, 쓰기 X, 실행 O
--- → 아무 권한 없음
```

> `x`를 준다고 평범한 텍스트 파일이 자동으로 정상 프로그램이 되는 것은 아니다. 실행 가능한 binary이거나 적절한 script 형식이어야 한다.

---

## 4. 왜 `r=4`, `w=2`, `x=1`인가?

이 부분이 숫자 Permission을 이해하는 핵심이다.

각 class의 Permission은 **3개의 bit**로 표현할 수 있다.

```text
r w x
1 1 1
```

Binary 자리값은:

```text
2² 2¹ 2⁰
4  2  1
```

그래서:

```text
r = 100₂ = 4
w = 010₂ = 2
x = 001₂ = 1
```

권한 조합은 bit가 켜진 자리값을 더한다.

| 권한 | Binary | 계산 | 숫자 |
|---|---:|---:|---:|
| `---` | `000` | 0 | 0 |
| `--x` | `001` | 1 | 1 |
| `-w-` | `010` | 2 | 2 |
| `-wx` | `011` | 2+1 | 3 |
| `r--` | `100` | 4 | 4 |
| `r-x` | `101` | 4+1 | 5 |
| `rw-` | `110` | 4+2 | 6 |
| `rwx` | `111` | 4+2+1 | 7 |

이 0~7 한 자리를 **8진수(octal) digit**로 표현한다.

---

## 5. 숫자 권한 계산

숫자 3자리는 순서대로:

```text
Owner | Group | Others
```

예: `640`

```text
6 → rw- → Owner
4 → r-- → Group
0 → --- → Others
```

따라서:

```text
640 = rw-r-----
```

### 자주 보는 Permission

| 숫자 | 문자 | 해석 |
|---:|---|---|
| `600` | `rw-------` | Owner만 읽기/쓰기 |
| `640` | `rw-r-----` | Owner rw, Group r |
| `644` | `rw-r--r--` | Owner rw, 나머지 r |
| `700` | `rwx------` | Owner만 rwx |
| `750` | `rwxr-x---` | Owner rwx, Group r-x |
| `755` | `rwxr-xr-x` | Owner rwx, 나머지 r-x |
| `777` | `rwxrwxrwx` | 모든 class에 rwx |

### 왜 `777`이 위험한가?

`777`은 Others에게도 write를 허용한다. 중요한 설정이나 공유 자원에서 필요 이상으로 권한을 넓히면 설정 변조·데이터 손상·보안 문제가 발생할 수 있다.

---

## 6. `ls -l` 출력 해석

```bash
ls -l app.conf
```

예:

```text
-rw-r----- 1 linuxuser ops 1024 Sep 7 10:00 app.conf
```

주요 부분:

```text
-           → 파일 타입: regular file
rw-r-----   → Permission
1           → link count
linuxuser   → Owner
ops         → Group
1024        → Size(bytes)
...
app.conf    → 이름
```

첫 10문자:

```text
- | rw- | r-- | ---
│   │     │     └─ Others
│   │     └─────── Group
│   └───────────── Owner
└───────────────── File type
```

주요 file type 문자:

```text
- = regular file
d = directory
l = symbolic link
```

### `ls -ld`

```bash
ls -ld /tmp
```

`-l` = long format
`-d` = directory 내용을 나열하지 않고 **directory 자체**를 표시

---

## 7. `chmod`

### 한 줄 정의

`chmod`는 **change mode**로 Permission bits를 변경한다.

```bash
chmod 640 app.conf
```

여기서:

```text
chmod → command
640   → 새 permission mode
app.conf → 대상 argument
```

`chmod`는 Owner 이름을 바꾸는 명령이 아니다.

```text
chmod → Permission 변경
chown → Ownership 변경
```

### 누가 `chmod`할 수 있나?

일반적으로 파일 Owner 또는 적절한 privilege를 가진 사용자가 Permission을 변경할 수 있다. 다른 사용자의 시스템 파일에는 `sudo`가 필요할 수 있다.

---

## 8. 🧪 실제 실습

```bash
touch permission-test.txt
ls -l permission-test.txt
```

변경:

```bash
chmod 600 permission-test.txt
chmod 644 permission-test.txt
chmod 755 permission-test.txt
chmod 640 permission-test.txt
```

각 단계에서:

```bash
ls -l permission-test.txt
```

로 확인했다.

직접 계산:

```text
600 → rw-------
644 → rw-r--r--
700 → rwx------
755 → rwxr-xr-x
640 → rw-r-----
750 → rwxr-x---
```

---

## 9. ⚠️ 헷갈리기 쉬운 부분

> ⚠️ `r=4`, `w=2`, `x=1`은 임의의 암기 숫자가 아니라 3-bit binary 자리값이다.

> ⚠️ `644`는 Group/Others가 아무 권한도 없는 것이 아니다. 둘 다 `r--`, 즉 읽기 가능하다.

> ⚠️ Permission과 Ownership은 다른 개념이다. `chmod`로 Owner를 바꿀 수 없다.

> ⚠️ 파일 Permission만 보고 접근 가능 여부를 항상 단정할 수 없다. 상위 Directory Permission, ACL, filesystem mount option 등 다른 조건도 영향을 줄 수 있다.

---

## 10. 🔧 Troubleshooting

### 증상

설정 파일이:

```text
-rwxrwxrwx
```

즉 `777`이었다.

요구사항:

```text
Owner = read/write
Group = read
Others = read
```

### 판단

필요한 mode:

```text
Owner  rw- = 6
Group  r-- = 4
Others r-- = 4
→ 644
```

### 조치

```bash
chmod 644 config.conf
```

### 검증

```bash
ls -l config.conf
```

예상:

```text
-rw-r--r--
```

### 운영 사고방식

`Permission denied`가 난다고 `chmod 777`부터 사용하지 않는다.

```text
현재 사용자 확인
→ Owner/Group 확인
→ 필요한 동작 확인
→ 부족한 Permission만 부여
→ 실제 접근 검증
```

---

## 11. 💼 실무 포인트

실무에서 Permission 문제는 보통 다음과 같이 본다.

```bash
whoami
id
ls -l file
ls -ld parent-directory
```

질문:

```text
누가 접근하는가?
어떤 Group에 속하는가?
파일 Owner/Group은 누구인가?
읽기/쓰기/실행 중 무엇이 필요한가?
상위 Directory를 통과할 수 있는가?
```

목표는 “권한을 크게 주는 것”이 아니라 **업무에 필요한 최소 권한을 정확히 주는 것**이다.

---

## 12. ✅ 핵심 정리

- 기본 Permission class는 Owner / Group / Others다.
- 파일 기준 `r=read`, `w=write`, `x=execute`다.
- `r=4`, `w=2`, `x=1`은 binary bit 자리값에서 나온다.
- 한 class의 Permission은 0~7의 octal 한 자리로 표현한다.
- `640 = rw-r-----`, `755 = rwxr-xr-x`다.
- `chmod`는 Permission bits를 변경한다.
- `777`을 임시 해결책처럼 사용하는 습관은 피한다.

---

## 13. 🧠 복습 문제

1. Owner / Group / Others는 각각 누구인가?
2. 왜 `r=4`, `w=2`, `x=1`인가?
3. `r-x`를 binary와 decimal/octal digit으로 표현해보라.
4. `chmod 640 file`을 문자 권한으로 바꿔보라.
5. `rwxr-x---`은 숫자로 몇인가?
6. `644`에서 Others는 무엇을 할 수 있는가?
7. `chmod`와 `chown`의 차이는?
8. Permission denied에서 `777`부터 주면 안 되는 이유는?
