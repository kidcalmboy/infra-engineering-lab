# Day 3 - 계정 잠금과 사용자 삭제

## 학습 목표

- Linux에서 **사용자 계정이 어떤 파일과 식별 정보로 관리되는지** 이해한다.
- `passwd -l`, `passwd -u`, `passwd -S`가 각각 무엇을 변경하거나 조회하는지 이해한다.
- **계정 잠금과 계정 삭제가 서로 다른 작업**이라는 점을 구분한다.
- `userdel`과 `userdel -r`의 차이와 삭제 후 남을 수 있는 데이터를 이해한다.
- 사용자 삭제 후 파일 소유자가 `1001` 같은 숫자로 표시되는 이유를 UID/GID 관점에서 설명할 수 있다.
- 운영 환경에서 계정을 바로 삭제하지 않고 **확인 → 잠금 → 데이터 점검 → 삭제 → 검증** 순서로 처리하는 이유를 이해한다.

---

# 1. Linux 사용자 계정은 무엇으로 구성되는가

Linux의 사용자 계정은 단순히 `username` 하나만 존재하는 것이 아니다.

한 명의 사용자는 여러 시스템 정보와 연결된다.

```text
사용자 계정
├─ 사용자 이름 (username)
├─ UID (User ID)
├─ 기본 GID (Group ID)
├─ 보조 그룹
├─ 홈 디렉터리
├─ 로그인 Shell
└─ 비밀번호 인증 정보
```

대표적으로 다음 파일들과 연결된다.

```text
/etc/passwd
→ 사용자 이름, UID, GID, 홈 디렉터리, 로그인 Shell 등 계정 기본 정보

/etc/shadow
→ 비밀번호 해시와 비밀번호 만료/잠금 관련 정보

/etc/group
→ 그룹 이름, GID, 그룹 구성원 정보
```

따라서 **사용자를 삭제한다는 것은 단순히 `/home/username` 디렉터리를 지우는 것과 다르다.**

사용자 계정 정보와 파일 데이터는 서로 연결되어 있지만 동일한 것은 아니다.

---

# 2. `/etc/passwd`와 `/etc/shadow`의 관계

사용자 기본 정보는 `/etc/passwd`에서 확인할 수 있다.

```bash
grep "^tempuser:" /etc/passwd
```

예:

```text
tempuser:x:1001:1001:,,,:/home/tempuser:/bin/bash
```

주요 의미:

```text
tempuser
→ 사용자 이름

x
→ 비밀번호 정보가 /etc/shadow에 별도로 저장되어 있음을 의미

1001
→ UID

1001
→ 기본 GID

/home/tempuser
→ 홈 디렉터리

/bin/bash
→ 로그인 Shell
```

실제 비밀번호 원문이 `/etc/passwd`에 저장되는 것은 아니다.

비밀번호 인증과 관련된 정보는 `/etc/shadow`에 저장된다.

```bash
sudo grep "^tempuser:" /etc/shadow
```

`/etc/shadow`에는 사용자가 입력한 비밀번호 문자열 자체가 아니라 **비밀번호를 검증하기 위한 해시 정보**와 비밀번호 만료 정책 등이 저장된다.

즉:

```text
/etc/passwd
→ 계정이 누구인지 설명하는 기본 정보

/etc/shadow
→ 그 계정의 비밀번호 인증 상태와 정책 정보
```

---

# 3. 계정 잠금이란 무엇인가

운영 환경에서는 사용자를 즉시 삭제하는 대신 먼저 **접근을 차단하고 데이터를 확인해야 하는 경우**가 있다.

예를 들어 퇴사자 계정이 있다고 가정한다.

바로 계정을 삭제하면 다음을 확인하기 어려워질 수 있다.

```text
- 홈 디렉터리에 업무 파일이 있는가?
- 다른 경로에도 해당 사용자가 만든 파일이 있는가?
- 해당 사용자가 어떤 그룹에 속해 있었는가?
- 서비스나 작업 스크립트가 그 UID를 사용하고 있는가?
```

따라서 상황에 따라 먼저 계정을 잠근 뒤 조사하는 방식이 안전하다.

```text
접근 차단
→ 데이터 확인
→ 필요한 자료 백업
→ 소유권 정리
→ 계정 삭제
```

---

# 4. `passwd -l` - 비밀번호 잠금

사용자의 비밀번호 인증을 잠글 때 사용한다.

```bash
sudo passwd -l tempuser
```

`-l`은 **lock**을 의미한다.

개념적으로는:

```text
사용자 계정 자체는 존재
/etc/passwd 정보도 존재
/home/tempuser도 존재
하지만 비밀번호를 이용한 인증은 잠김
```

Linux에서는 일반적으로 `/etc/shadow`에 저장된 비밀번호 해시 앞에 잠금 표시를 추가하는 방식으로 비밀번호 인증을 사용할 수 없게 만든다.

중요한 점은:

> `passwd -l`은 **사용자 계정을 삭제하는 명령이 아니다.**

따라서 다음 명령은 여전히 사용자를 찾을 수 있다.

```bash
id tempuser
```

```bash
grep "^tempuser:" /etc/passwd
```

## 계정 잠금의 중요한 한계

`passwd -l`은 **비밀번호 기반 인증을 잠그는 것**이다.

따라서 환경에 따라 SSH 공개키 등 다른 인증 방식이 설정되어 있다면, 이것만으로 모든 형태의 로그인을 완전히 차단했다고 단정하면 안 된다.

운영 환경에서는 필요에 따라 SSH 키, 로그인 Shell, 계정 만료 정책 등 다른 접근 경로도 함께 점검해야 한다.

현재 실습에서는 우선 다음처럼 이해한다.

```text
passwd -l
→ 비밀번호 인증 잠금
```

---

# 5. `passwd -u` - 비밀번호 잠금 해제

잠긴 비밀번호 인증을 다시 사용할 수 있도록 해제한다.

```bash
sudo passwd -u tempuser
```

`-u`는 **unlock**을 의미한다.

```text
passwd -l
→ 잠금

passwd -u
→ 잠금 해제
```

계정을 다시 활성 상태로 돌릴 때 사용한다.

---

# 6. `passwd -S` - 비밀번호 상태 확인

계정의 비밀번호 상태를 조회할 때 사용한다.

```bash
sudo passwd -S tempuser
```

`-S`는 **Status**를 의미한다.

예:

```text
tempuser L 2026-09-01 0 99999 7 -1
```

각 항목은 비밀번호 상태와 비밀번호 수명 정책을 나타낸다.

```text
tempuser
→ 사용자 이름

L
→ 현재 비밀번호 상태

2026-09-01
→ 마지막 비밀번호 변경일

0
→ 비밀번호를 다시 변경할 수 있기까지의 최소 일수

99999
→ 비밀번호 최대 사용 일수

7
→ 비밀번호 만료 전 경고 일수

-1
→ 비밀번호 만료 후 비활성화 관련 값
```

현재 단계에서 가장 중요한 것은 두 번째 필드다.

```text
P
→ Password set
→ 사용할 수 있는 비밀번호가 설정된 상태

L
→ Locked
→ 비밀번호가 잠긴 상태

NP
→ No Password
→ 비밀번호가 설정되지 않은 상태
```

따라서 실습에서:

```bash
sudo passwd -l tempuser
sudo passwd -S tempuser
```

실행 후:

```text
tempuser L ...
```

이 보였다면 잠금이 정상적으로 적용된 것이다.

잠금 해제 후:

```bash
sudo passwd -u tempuser
sudo passwd -S tempuser
```

다시 상태를 비교하면 된다.

즉 `passwd -S`는 **변경 명령이 아니라 상태 확인 명령**이다.

---

# 7. 계정 잠금과 계정 삭제의 차이

둘은 목적부터 다르다.

```text
계정 잠금
→ 계정 정보와 데이터는 유지
→ 접근을 제한

계정 삭제
→ 사용자 계정 자체를 시스템에서 제거
```

예를 들어:

```bash
sudo passwd -l tempuser
```

후에는:

```bash
id tempuser
```

가 여전히 사용자를 찾을 수 있다.

반면:

```bash
sudo userdel tempuser
```

후에는:

```bash
id tempuser
```

에서 사용자가 존재하지 않는다는 결과가 나와야 한다.

---

# 8. `userdel` - 사용자 계정 삭제

사용자 계정을 시스템에서 삭제한다.

```bash
sudo userdel tempuser
```

여기서 중요한 점은 **계정과 홈 디렉터리를 별개의 대상으로 생각해야 한다는 것**이다.

기본 `userdel`은 계정을 삭제하지만 홈 디렉터리를 그대로 남길 수 있다.

```text
sudo userdel tempuser

/etc/passwd의 tempuser 계정
→ 삭제

/etc/shadow의 tempuser 인증 정보
→ 삭제

/home/tempuser
→ 남을 수 있음
```

따라서:

```bash
sudo userdel tempuser
```

실행 후 `/home/tempuser`가 존재한다고 해서 사용자 삭제가 실패했다고 판단하면 안 된다.

계정 존재 여부는 다음처럼 확인한다.

```bash
id tempuser
```

또는:

```bash
grep "^tempuser:" /etc/passwd
```

---

# 9. `userdel -r` - 홈 디렉터리까지 제거

계정과 홈 디렉터리를 함께 제거하려면:

```bash
sudo userdel -r tempuser
```

`-r` 옵션을 사용한다.

현재 단계에서는 다음처럼 구분하면 된다.

```text
userdel tempuser
→ 사용자 계정 삭제
→ 홈 디렉터리는 남을 수 있음

userdel -r tempuser
→ 사용자 계정 삭제
→ 홈 디렉터리 및 관련 사용자 데이터 일부도 제거
```

`-r`은 데이터까지 제거할 수 있기 때문에 운영 서버에서는 특히 주의해야 한다.

## `-r`을 사용해도 모든 파일이 자동으로 사라지는 것은 아니다

사용자가 홈 디렉터리 밖에 파일을 만들었을 수도 있다.

예:

```text
/var/app/data/report.txt
/shared/project/file.txt
```

이런 파일이 `tempuser` 소유라면 계정을 삭제한 뒤에도 남을 수 있다.

따라서:

> `userdel -r` = 그 사용자가 시스템에 만든 모든 파일을 완전히 제거

라고 이해하면 안 된다.

---

# 10. 계정 삭제 후 `1001 1001`처럼 보이는 이유

실습에서 `tempuser`를 삭제하고 홈 디렉터리를 확인했을 때:

```bash
ls -ld /home/tempuser
```

다음처럼 표시될 수 있다.

```text
drwxr-x--- 2 1001 1001 4096 ... /home/tempuser
```

왜 `tempuser tempuser`가 아니라 `1001 1001`로 보일까?

Linux 파일시스템의 소유권은 실제로 **사용자 이름 문자열이 아니라 UID와 GID 숫자로 저장되기 때문**이다.

사용자가 존재할 때:

```text
파일에 저장된 실제 소유권
UID 1001
GID 1001

/etc/passwd와 /etc/group 조회
↓
UID 1001 → tempuser
GID 1001 → tempuser

ls 출력
↓
tempuser tempuser
```

사용자를 삭제하면 UID와 사용자 이름의 매핑이 사라진다.

```text
파일에는 여전히 UID 1001 저장
하지만 /etc/passwd에 UID 1001 사용자 없음
↓
이름으로 변환 불가
↓
ls가 1001 숫자를 그대로 표시
```

즉 파일의 소유권 값 자체가 사라진 것이 아니다.

**이름으로 보여줄 계정 정보가 없어졌기 때문에 숫자가 보이는 것**이다.

---

# 11. 왜 남은 UID/GID 파일을 확인해야 하는가

삭제된 사용자가 소유하던 파일이 남아 있다면 이를 **orphaned file(소유자 계정이 사라진 파일)** 관점에서 점검해야 한다.

운영자는 필요에 따라:

```text
- 파일이 필요한 데이터인지 확인
- 다른 사용자나 서비스 계정으로 소유권 이전
- 백업
- 불필요하다면 삭제
```

등을 판단해야 한다.

또한 UID는 나중에 새로운 사용자에게 재사용될 가능성도 있으므로, 중요한 서버에서는 계정 삭제 전후의 소유 파일을 확인하는 습관이 중요하다.

---

# 12. 사용자 삭제 전 무엇을 확인해야 하는가

삭제부터 하지 않는다.

우선 계정 상태를 확인한다.

```bash
id tempuser
```

계정 정보:

```bash
grep "^tempuser:" /etc/passwd
```

그룹:

```bash
groups tempuser
```

홈 디렉터리:

```bash
ls -ld /home/tempuser
```

홈 디렉터리 내부:

```bash
ls -al /home/tempuser
```

필요하면 계정을 먼저 잠근다.

```bash
sudo passwd -l tempuser
```

상태 확인:

```bash
sudo passwd -S tempuser
```

그 후 데이터를 백업하거나 소유권을 정리하고 계정을 삭제한다.

---

# 13. 운영 관점의 안전한 계정 정리 절차

실무적인 사고 흐름은 다음과 같다.

```text
1. 대상 사용자 확인
        ↓
2. UID / GID / 그룹 확인
        ↓
3. 필요하면 계정 접근 잠금
        ↓
4. 홈 디렉터리와 업무 데이터 확인
        ↓
5. 필요한 데이터 백업
        ↓
6. 남겨야 할 파일의 소유권 처리
        ↓
7. 사용자 계정 삭제
        ↓
8. 홈 디렉터리 또는 잔여 파일 확인
        ↓
9. 최종 검증
```

핵심은:

> **삭제 명령을 아는 것보다 삭제 전에 무엇을 확인해야 하는지 아는 것이 더 중요하다.**

---

# 14. 실습

실습 사용자 생성:

```bash
sudo adduser tempuser
```

계정 확인:

```bash
id tempuser
ls -ld /home/tempuser
```

계정 잠금:

```bash
sudo passwd -l tempuser
```

상태 확인:

```bash
sudo passwd -S tempuser
```

출력의 두 번째 필드가:

```text
L
```

이면 잠금 상태다.

잠금 해제:

```bash
sudo passwd -u tempuser
```

계정만 삭제:

```bash
sudo userdel tempuser
```

계정 삭제 확인:

```bash
id tempuser
```

홈 디렉터리 확인:

```bash
ls -ld /home/tempuser
```

실습에서는:

```text
계정
→ 삭제됨

/home/tempuser
→ 남아 있음

소유자 표시
→ 1001 1001
```

상태를 직접 확인했다.

---

# 15. Troubleshooting - `userdel` 후 홈 디렉터리가 남아 있음

## 상황

운영자가 다음 명령을 실행했다.

```bash
sudo userdel tempuser
```

그런데:

```bash
ls -ld /home/tempuser
```

를 실행하니 홈 디렉터리가 그대로 남아 있다.

초보 운영자는:

```text
사용자 삭제가 실패했다.
```

라고 판단할 수 있다.

## 확인 1 - 계정이 실제로 존재하는가

```bash
id tempuser
```

사용자가 없다고 나오면 계정 자체는 삭제된 것이다.

## 확인 2 - `/etc/passwd` 확인

```bash
grep "^tempuser:" /etc/passwd
```

아무 결과가 없다면 계정 정보가 제거된 것이다.

## 확인 3 - 홈 디렉터리 확인

```bash
ls -ld /home/tempuser
```

홈 디렉터리는 별도의 파일시스템 객체이므로 남아 있을 수 있다.

## Root Cause

```text
userdel의 기본 동작은
사용자 계정 삭제이며,
홈 디렉터리까지 반드시 삭제하는 것은 아니다.
```

따라서:

```text
계정 없음
+
/home/tempuser 존재
```

는 모순이 아니며 정상적으로 발생할 수 있다.

## Resolution

처음부터 홈 디렉터리까지 제거하는 것이 요구사항이었다면:

```bash
sudo userdel -r tempuser
```

을 사용한다.

이미 계정을 삭제한 뒤 홈 디렉터리만 남았다면 데이터를 확인하고, 정말 필요 없을 때 별도로 제거한다.

## Verification

```bash
id tempuser
ls -ld /home/tempuser
```

계정과 파일시스템을 각각 따로 확인한다.

---

# 16. 이번 챕터 핵심 명령어

```bash
# 비밀번호 잠금
sudo passwd -l username

# 비밀번호 잠금 해제
sudo passwd -u username

# 비밀번호 상태 확인
sudo passwd -S username

# 사용자 계정 삭제
sudo userdel username

# 사용자 + 홈 디렉터리 제거
sudo userdel -r username

# 사용자 존재 확인
id username

# 계정 정보 확인
grep "^username:" /etc/passwd

# 홈 디렉터리 자체 정보 확인
ls -ld /home/username
```

---

# 17. 반드시 이해해야 할 핵심 개념

```text
passwd -l
→ 사용자 삭제가 아니라 비밀번호 인증 잠금

passwd -u
→ 비밀번호 인증 잠금 해제

passwd -S
→ 비밀번호 상태와 비밀번호 수명 정책 조회

P
→ Password set

L
→ Locked

NP
→ No Password

userdel
→ 사용자 계정 삭제
→ 홈 디렉터리는 남을 수 있음

userdel -r
→ 계정과 홈 디렉터리를 함께 제거
→ 홈 디렉터리 밖의 모든 소유 파일까지 자동 삭제하는 것은 아님

삭제 후 1001 같은 숫자가 보이는 이유
→ 파일 소유권은 UID/GID로 저장됨
→ 계정 삭제로 이름 매핑이 없어져 숫자가 그대로 보임

운영 핵심
→ 확인 → 잠금 → 데이터 점검/백업 → 삭제 → 잔여 데이터 확인 → 검증
```

---

# 학습 포인트

계정 관리에서 가장 중요한 것은 명령어를 외우는 것이 아니다.

```text
계정 정보와 데이터는 같은 것이 아니다.
사용자 이름과 UID도 같은 것이 아니다.
계정 잠금과 계정 삭제도 같은 것이 아니다.
```

이 차이를 이해해야 사용자 삭제 후 홈 디렉터리가 남거나 소유자가 숫자로 보이는 상황을 장애로 오해하지 않고 정확히 판단할 수 있다.
