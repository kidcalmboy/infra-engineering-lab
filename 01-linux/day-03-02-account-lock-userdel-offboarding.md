# Day 3-2 — 계정 잠금·상태 확인·사용자 삭제

> 사용자 계정을 바로 삭제하지 않고 **상태 확인 → 접근 제한 → 데이터 확인 → 삭제 → 잔여 파일 검증** 순서로 처리하는 운영 절차를 학습했다. 핵심은 계정 정보, 인증 방식, 파일 소유권을 서로 분리해서 이해하는 것이다.

## 📌 이번에 배운 내용

- 계정(Account)과 인증(Authentication)의 차이
- `passwd -l`, `-u`, `-S`
- `P`, `L`, `NP` 상태
- `/etc/shadow`와 password lock
- password lock과 SSH public key 인증의 차이
- `userdel`, `userdel -r`
- 파일 소유권에 UID/GID가 저장되는 방식
- 계정 삭제 후 숫자 UID/GID가 남는 이유
- 안전한 Account Lifecycle

## 📚 목차

1. 계정과 인증
2. Password Lock
3. `passwd` 주요 옵션
4. 상태 값
5. `userdel`
6. UID/GID 잔여 파일
7. 실제 실습
8. 헷갈리기 쉬운 부분
9. Troubleshooting
10. 실무 포인트
11. 핵심 정리
12. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 옵션 | 의미 |
|---|---|---|
| `passwd -l user` | `-l` = lock | password 인증 잠금 |
| `passwd -u user` | `-u` = unlock | password 잠금 해제 |
| `passwd -S user` | `-S` = status | password 상태 확인 |
| `id user` | 사용자명 | 계정 존재 및 UID/GID 확인 |
| `userdel user` | 없음 | 계정 DB 항목 삭제 |
| `userdel -r user` | `-r` = remove | 계정 + home/mail spool 등 제거 |
| `ls -ld /home/user` | `-l`, `-d` | home 존재/소유권 확인 |
| `find / -uid UID` | `-uid` | 특정 UID 소유 파일 검색 |

---

## 1. 계정과 인증은 같은 개념이 아니다

### Account

Linux 시스템에 등록된 사용자 Identity다. Username, UID, Primary GID, Home, Login Shell 같은 정보가 있다.

### Authentication

사용자가 자신이 누구인지 증명하는 과정이다.

가능한 방식:

```text
Password
SSH Public Key
Kerberos
PAM 연동
기타 외부 인증
```

따라서 **password를 잠그는 것과 계정 전체의 모든 로그인 경로를 차단하는 것은 동일하지 않다.**

---

## 2. Password Lock이란?

`passwd -l`은 주로 `/etc/shadow`의 password hash field를 잠긴 형태로 변경하여 password 기반 인증을 막는다.

```bash
sudo passwd -l tempuser
```

이것은:

```text
계정 삭제 X
Home 삭제 X
UID 삭제 X
Password 인증 제한 O
```

에 가깝다.

SSH public key 같은 다른 인증 방식이 허용되어 있다면 추가 확인이 필요하다.

---

## 3. `passwd` 주요 옵션

### `-l` — lock

```bash
sudo passwd -l tempuser
```

password 인증을 잠근다.

### `-u` — unlock

```bash
sudo passwd -u tempuser
```

잠금 상태를 해제한다.

### `-S` — status

```bash
sudo passwd -S tempuser
```

Password 상태와 aging 정보를 요약해서 보여준다.

### 자주 보게 될 추가 옵션

- `-d`: password 삭제
- `-e`: password 만료 처리, 다음 로그인에서 변경 유도 가능

운영에서는 옵션의 효과를 이해하지 못한 상태에서 계정 정책을 변경하지 않는다.

---

## 4. `passwd -S` 상태 값

대표적으로:

```text
P  = usable Password가 설정된 상태
L  = Locked
NP = No Password
```

예:

```text
tempuser L 2026-09-03 0 99999 7 -1
```

뒤의 숫자는 배포판/도구 형식에 따라 마지막 password 변경일, 최소/최대 사용일, 경고 기간, 비활성 기간 등 aging 정보와 연결된다.

현재 단계 핵심은 `P/L/NP`를 구분하고 **상태를 확인한 뒤 조치한다**는 점이다.

---

## 5. `userdel`

### `userdel user`

```bash
sudo userdel tempuser
```

사용자 계정 정보를 삭제한다. Home directory가 자동으로 지워진다고 가정하면 안 된다.

### `userdel -r user`

```bash
sudo userdel -r tempuser
```

`-r`은 Home directory와 mail spool 등 관련 사용자 데이터를 함께 제거하도록 요청한다.

중요:

> `-r`을 사용해도 시스템 전체에 흩어진 모든 파일을 반드시 제거한다고 생각하면 안 된다.

사용자가 `/srv`, `/opt`, 공유 디렉터리 등에 만든 파일은 별도로 남을 수 있다.

---

## 6. 계정 삭제 후 숫자 UID/GID가 남는 이유

Linux 파일 inode metadata에는 보통 Username 문자열이 아니라 **UID/GID 숫자**가 저장된다.

계정이 있을 때:

```text
파일 metadata UID=1001
        ↓
시스템이 UID 1001을 조회
        ↓
tempuser라는 이름으로 표시
```

계정 삭제 후:

```text
파일 metadata UID=1001은 그대로
        ↓
UID 1001에 매핑되는 username 없음
        ↓
ls가 1001을 숫자로 표시
```

즉 파일 ownership이 계정 삭제와 함께 자동으로 사라지는 것이 아니다.

### 왜 위험할 수 있나?

나중에 UID 1001이 다른 사용자에게 재사용되면 과거 파일이 새 사용자 소유처럼 보일 수 있어 관리가 복잡해질 수 있다.

그래서 계정 삭제 전후로 해당 UID의 파일을 확인하는 것이 중요하다.

예:

```bash
find / -uid 1001 2>/dev/null
```

`-uid`는 숫자 UID를 기준으로 검색한다.

---

## 7. 🧪 실제 실습

계정 생성:

```bash
sudo adduser tempuser
id tempuser
ls -ld /home/tempuser
```

잠금:

```bash
sudo passwd -l tempuser
sudo passwd -S tempuser
```

`L` 상태 확인.

잠금 해제:

```bash
sudo passwd -u tempuser
sudo passwd -S tempuser
```

계정만 삭제:

```bash
sudo userdel tempuser
```

검증:

```bash
id tempuser
ls -ld /home/tempuser
```

계정은 사라졌지만 Home이 남고 Owner/Group이 숫자 UID/GID로 보일 수 있음을 확인했다.

---

## 8. ⚠️ 헷갈리기 쉬운 부분

> ⚠️ `passwd -l`은 모든 인증 방식을 완전히 차단하는 명령으로 단정하면 안 된다.

> ⚠️ `userdel` 성공과 Home directory 삭제 여부는 별개다.

> ⚠️ `userdel -r`도 Home 밖의 모든 사용자 파일을 완전히 제거한다고 보장하는 의미는 아니다.

> ⚠️ 계정 이름이 사라져도 파일에는 UID/GID 숫자가 남아 있을 수 있다.

---

## 9. 🔧 Troubleshooting

### 증상

```bash
sudo userdel tempuser
```

후에도 `/home/tempuser`가 남아 있다.

### 확인

```bash
id tempuser
ls -ld /home/tempuser
```

### 판단

```text
id 실패 → 계정은 삭제됨
Home 존재 → 데이터 디렉터리는 별도로 남음
```

### 조치

중요 데이터 여부를 확인하고 백업/이관 후 필요한 경우 안전하게 제거한다.

운영에서는:

```text
Account 존재 여부
Home 존재 여부
프로세스 존재 여부
소유 파일 존재 여부
Authentication 경로
```

를 각각 독립적으로 확인한다.

---

## 10. 💼 실무 포인트

안전한 퇴사자/사용 종료 계정 절차 예:

```text
1. 사용자/UID/GID 확인
2. 필요 시 접근 잠금
3. 활성 Session/Process 확인
4. SSH Key 등 다른 인증 경로 확인
5. Home 및 업무 데이터 확인
6. 데이터 백업/소유권 이관
7. UID 기준 잔여 파일 검색
8. 계정 삭제
9. Home/잔여 파일 정리
10. 최종 검증
```

운영에서 중요한 것은 삭제 명령 자체보다 **Identity와 Data Lifecycle을 안전하게 마감하는 것**이다.

---

## 11. ✅ 핵심 정리

- Account와 Authentication은 같은 개념이 아니다.
- `passwd -l`은 password 인증 잠금이다.
- `passwd -S`의 대표 상태는 P/L/NP다.
- `userdel`은 계정 정보를 삭제하며 Home은 남을 수 있다.
- `userdel -r`은 Home 등도 제거하지만 시스템 전체 사용자 파일까지 모두 지운다고 단정하면 안 된다.
- 파일 ownership은 UID/GID 숫자로 저장되므로 계정 삭제 후 숫자만 보일 수 있다.
- 계정 삭제 전후에는 데이터와 잔여 파일을 확인해야 한다.

---

## 12. 🧠 복습 문제

1. Account와 Authentication의 차이는?
2. `passwd -l`이 SSH key 인증까지 자동 차단한다고 볼 수 없는 이유는?
3. `P`, `L`, `NP`는 각각 무엇인가?
4. `userdel`과 `userdel -r`의 차이는?
5. 계정 삭제 후 파일 Owner가 숫자로 보이는 이유는?
6. UID 재사용이 왜 문제가 될 수 있는가?
7. 계정 삭제 전 어떤 순서로 데이터를 확인해야 하는가?
