# Day 4-1 - Linux 파일 권한 기초와 chmod

## 학습 목표

- Linux에서 파일 권한이 왜 필요한지 이해한다.
- `ls -l`, `ls -ld` 출력에서 파일 종류, 권한, 소유자, 소유 그룹을 구분한다.
- `Owner`, `Group`, `Others`가 각각 누구를 의미하는지 이해한다.
- `r`, `w`, `x` 권한의 의미를 파일 기준으로 설명할 수 있다.
- 숫자 권한에서 `r=4`, `w=2`, `x=1`이 어떻게 조합되는지 이해한다.
- `chmod`로 파일 권한을 설정하고 변경 결과를 검증한다.
- `777`처럼 과도하게 열린 권한이 왜 위험한지 운영 관점에서 판단한다.

---

## 1. Linux에서 파일 권한이 필요한 이유

Linux는 여러 사용자가 동시에 사용할 수 있는 다중 사용자 운영체제다.

서버 안에는 시스템 설정 파일, 로그, 사용자 데이터, 서비스 실행 파일 등 다양한 파일이 존재한다. 모든 사용자가 모든 파일을 자유롭게 읽고 수정하거나 실행할 수 있다면 설정 훼손, 정보 노출, 서비스 장애 같은 문제가 발생할 수 있다.

그래서 Linux는 파일과 디렉터리에 대해 다음 세 가지를 함께 관리한다.

```text
누가 소유하고 있는가?
누가 같은 그룹에 속해 있는가?
각 사용자가 무엇을 할 수 있는가?
```

기본 권한 구조는 다음과 같다.

```text
Owner  → 파일 소유자
Group  → 파일 소유 그룹에 속한 사용자
Others → 위 두 범주에 포함되지 않는 나머지 사용자
```

각 범주에는 다시 다음 권한이 적용된다.

```text
r = read
w = write
x = execute
```

즉 Linux 기본 권한은 다음 구조로 생각하면 된다.

```text
사용자/그룹 구조
Owner / Group / Others
        ↓
행동 권한
Read / Write / Execute
```

Day 3에서 배운 사용자(User), 그룹(Group), UID, GID가 Day 4의 파일 권한과 연결된다.

---

## 2. 파일 권한 확인 - `ls -l`

실습 파일을 만든다.

```bash
cd ~/linux-test
touch permission-test.txt
ls -l permission-test.txt
```

예시 출력:

```text
-rw-r--r-- 1 linuxuser linuxuser 0 Sep 3 10:00 permission-test.txt
```

이 출력은 단순히 파일 이름만 보여주는 것이 아니라 파일의 권한과 소유권 정보를 함께 보여준다.

핵심 부분은 다음과 같다.

```text
-rw-r--r--  linuxuser  linuxuser
│           │          │
│           │          └─ 소유 그룹(Group)
│           └──────────── 소유 사용자(Owner)
└──────────────────────── 파일 종류 + 권한
```

---

## 3. `ls -ld`는 무엇인가

`ls -ld`는 두 옵션이 결합된 형태다.

```bash
ls -ld /home/testuser
```

### `-l`

`l`은 long format의 의미로 이해하면 된다.

파일이나 디렉터리의 상세 정보를 출력한다.

```text
권한
링크 수
소유자
소유 그룹
크기
수정 시각
파일/디렉터리 이름
```

### `-d`

`d`는 디렉터리의 **내용이 아니라 디렉터리 자체**를 대상으로 출력하게 한다.

예를 들어:

```bash
ls -l /home/testuser
```

는 `/home/testuser` 안의 파일과 디렉터리를 보여준다.

반면:

```bash
ls -ld /home/testuser
```

는 `/home/testuser`라는 디렉터리 자체의 권한과 소유자 정보를 보여준다.

권한 문제를 확인할 때 디렉터리 자체 권한을 보고 싶다면 `ls -ld`가 유용하다.

---

## 4. 권한 문자열 구조 읽기

예:

```text
-rw-r--r--
```

총 10개의 문자를 다음과 같이 나눈다.

```text
- rw- r-- r--
│ │   │   │
│ │   │   └─ Others 권한
│ │   └───── Group 권한
│ └───────── Owner 권한
└─────────── 파일 종류
```

첫 번째 문자는 권한이 아니라 파일 종류를 나타낸다.

대표적으로:

```text
- = 일반 파일
d = 디렉터리
l = 심볼릭 링크
```

예를 들어 디렉터리는 다음처럼 시작할 수 있다.

```text
drwxr-xr-x
```

여기서 첫 번째 `d`는 directory라는 뜻이다.

---

## 5. Owner / Group / Others

Linux 기본 권한은 사용자 개개인마다 별도로 한 줄씩 권한을 지정하는 방식이 아니라 크게 세 범주로 나눈다.

### Owner

파일을 소유한 사용자다.

예:

```text
linuxuser
```

### Group

해당 파일이 연결된 소유 그룹이다.

그 그룹에 속한 사용자는 Group 영역의 권한을 적용받을 수 있다.

### Others

Owner도 아니고 해당 Group에도 해당하지 않는 나머지 사용자다.

따라서 다음 권한:

```text
rw-r-----
```

은 다음 의미다.

```text
Owner  → rw-
Group  → r--
Others → ---
```

즉:

```text
소유자      → 읽기 + 쓰기
같은 그룹   → 읽기만
그 외 사용자 → 아무 권한 없음
```

---

## 6. 파일에서 `r`, `w`, `x` 의미

### `r` - Read

파일 내용을 읽을 수 있는 권한이다.

예:

```bash
cat file.txt
less file.txt
```

### `w` - Write

파일 내용을 수정할 수 있는 권한이다.

예:

```text
기존 내용 변경
내용 추가
파일을 편집기로 수정
```

### `x` - Execute

파일을 실행할 수 있는 권한이다.

예를 들어 Shell Script가 있다면:

```bash
./script.sh
```

형태로 실행하려면 실행 권한이 필요하다.

다만 실행 권한을 준다고 일반 텍스트 파일이 자동으로 프로그램으로 변하는 것은 아니다.

```text
chmod +x
→ 실행을 허용하는 권한을 부여

파일 형식 자체
→ 그대로 유지
```

실행 가능한 내용이 들어 있는 파일에 `x` 권한이 의미가 있다.

---

## 7. `chmod`란 무엇인가

`chmod`는 **change mode**의 의미로 이해하면 된다.

파일이나 디렉터리의 접근 권한을 변경할 때 사용한다.

예:

```bash
chmod 640 permission-test.txt
```

이 명령은 `permission-test.txt`에 대해 다음 권한을 설정한다.

```text
Owner  → rw-
Group  → r--
Others → ---
```

즉 `chmod`는 단순히 "파일을 잠그는 명령"이 아니라 다음을 결정하는 명령이다.

```text
소유자에게 어떤 권한을 줄 것인가?
소유 그룹에게 어떤 권한을 줄 것인가?
나머지 사용자에게 어떤 권한을 줄 것인가?
```

주의할 점은 `chmod`가 **소유자를 바꾸는 명령은 아니라는 것**이다.

```text
chmod
→ 권한 변경

chown
→ 소유자 변경
```

`chown`은 이후 학습한다.

---

## 8. 숫자 권한의 원리

Linux에서는 `r`, `w`, `x`를 숫자로 표현할 수 있다.

```text
r = 4
w = 2
x = 1
```

각 범주의 권한을 더해서 하나의 숫자를 만든다.

```text
rwx = 4 + 2 + 1 = 7
rw- = 4 + 2     = 6
r-x = 4 + 1     = 5
r-- = 4         = 4
-wx = 2 + 1     = 3
-w- = 2         = 2
--x = 1         = 1
--- = 0         = 0
```

세 자리 권한은 각각 다음 대상을 의미한다.

```text
chmod 640 file.txt
      │││
      ││└─ Others
      │└── Group
      └─── Owner
```

따라서 `640`은:

```text
6 = rw-
4 = r--
0 = ---
```

즉:

```text
Owner  → rw-
Group  → r--
Others → ---
```

이다.

---

## 9. 자주 보는 숫자 권한

```text
600 → rw-------
644 → rw-r--r--
640 → rw-r-----
700 → rwx------
750 → rwxr-x---
755 → rwxr-xr-x
777 → rwxrwxrwx
```

숫자를 무작정 외우기보다 `4 + 2 + 1` 원리를 이용해 문자 권한으로 변환할 수 있어야 한다.

---

## 10. 직접 실습

### 파일 생성과 초기 권한 확인

```bash
cd ~/linux-test
touch permission-test.txt
ls -l permission-test.txt
```

### 600으로 변경

```bash
chmod 600 permission-test.txt
ls -l permission-test.txt
```

예상:

```text
-rw-------
```

### 644로 변경

```bash
chmod 644 permission-test.txt
ls -l permission-test.txt
```

예상:

```text
-rw-r--r--
```

### 755로 변경

```bash
chmod 755 permission-test.txt
ls -l permission-test.txt
```

예상:

```text
-rwxr-xr-x
```

일반 텍스트 파일에 `x` 권한이 추가됐더라도 파일 자체가 실행 가능한 프로그램으로 변한 것은 아니다.

---

## 11. 실습 문제와 정답

### 숫자 권한 → 문자 권한

```text
600 → rw-------
644 → rw-r--r--
700 → rwx------
755 → rwxr-xr-x
```

`ls -l`에서 일반 파일로 보인다면 앞에 파일 종류 `-`가 붙어서 다음처럼 보일 수 있다.

```text
-rw-------
-rw-r--r--
-rwx------
-rwxr-xr-x
```

### 문자 권한 → 숫자 권한

```text
rw-r----- → 640
rwxr-x--- → 750
rw------- → 600
rwxr-xr-x → 755
```

### 요구사항에 맞는 권한 설정

요구사항:

```text
Owner  → 읽기 + 쓰기
Group  → 읽기
Others → 권한 없음
```

계산:

```text
Owner  rw- = 6
Group  r-- = 4
Others --- = 0
```

명령:

```bash
chmod 640 permission-test.txt
```

---

## 12. Troubleshooting - 중요한 설정 파일이 777인 경우

### 상황

중요한 서버 설정 파일의 권한이 다음과 같다.

```text
-rwxrwxrwx
```

숫자로는:

```text
777
```

### 현재 상태 해석

```text
Owner  → rwx
Group  → rwx
Others → rwx
```

즉 Owner뿐만 아니라 Group과 Others까지 파일을 수정할 수 있다.

중요한 설정 파일을 불필요한 사용자가 수정할 수 있으면 다음 문제가 발생할 수 있다.

```text
설정 변조
서비스 오동작
보안 설정 변경
장애 발생
권한 오남용
```

핵심은 "모든 사람이 파일을 볼 수 있어서 위험하다"가 아니라 **업무상 필요하지 않은 사용자에게 쓰기·실행 권한까지 열려 있다는 것**이다.

### 요구 권한

```text
Owner  → 읽기 + 쓰기
Group  → 읽기
Others → 읽기
```

계산:

```text
Owner  rw- = 6
Group  r-- = 4
Others r-- = 4
```

따라서:

```text
644
```

가 된다.

### 조치

```bash
chmod 644 파일이름
```

### 검증

```bash
ls -l 파일이름
```

기대 결과:

```text
-rw-r--r--
```

여기서 `644`는 Group과 Others의 **읽기 권한은 허용**한다.

따라서:

```text
644 = 다른 사용자는 접근 불가
```

가 아니라:

```text
644 = 다른 사용자는 읽을 수 있지만 수정하거나 실행할 수 없음
```

이라고 이해해야 한다.

---

## 13. 운영 관점 핵심

권한 문제를 확인할 때는 다음 흐름으로 접근한다.

```text
대상 확인
→ ls -l 또는 ls -ld로 현재 권한 확인
→ Owner / Group / Others 분리
→ 필요한 업무 권한 판단
→ chmod로 최소 권한만 부여
→ ls -l로 결과 검증
```

중요한 운영 원칙은 **최소 권한 원칙(Principle of Least Privilege)** 이다.

```text
업무에 필요한 권한만 부여하고
필요하지 않은 권한은 열지 않는다.
```

편하다는 이유로 `777`을 사용하는 습관은 피해야 한다.

---

## 핵심 정리

```text
ls -l
→ 파일 상세 정보 확인

ls -ld
→ 디렉터리 내부가 아니라 디렉터리 자체 상세 정보 확인

Owner
→ 파일 소유 사용자

Group
→ 파일 소유 그룹

Others
→ Owner와 Group을 제외한 나머지 사용자

r
→ 파일 읽기

w
→ 파일 수정

x
→ 파일 실행 허용

chmod
→ 파일/디렉터리 권한 변경

r = 4
w = 2
x = 1

640
→ rw-r-----

644
→ rw-r--r--

755
→ rwxr-xr-x

777
→ rwxrwxrwx
→ 필요 이상으로 권한이 열릴 수 있으므로 주의
```

## 다음 학습

다음 단계에서는 **디렉터리에서 `r`, `w`, `x`가 어떤 의미를 가지는지** 학습한다.

파일과 디렉터리는 같은 `rwx` 문자를 사용하지만 실제 의미가 다르기 때문에 반드시 구분해야 한다.
