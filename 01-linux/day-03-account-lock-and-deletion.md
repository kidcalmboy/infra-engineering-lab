# Day 3 - 계정 잠금과 사용자 삭제

## 학습 목표

- `passwd -l`, `passwd -u`, `passwd -S`로 계정 비밀번호 상태를 관리한다.
- `userdel`과 `userdel -r`의 차이를 이해한다.
- 사용자 삭제 후 홈 디렉터리와 UID/GID가 어떻게 남는지 확인한다.
- 계정을 바로 삭제하기보다 잠금 → 확인/백업 → 삭제 순서로 접근하는 운영 습관을 익힌다.

## 계정 잠금 - `passwd -l`

사용자의 비밀번호 기반 로그인을 잠글 때 사용한다.

```bash
sudo passwd -l tempuser
```

`-l`은 lock을 의미한다.

```text
계정 자체는 유지
→ 비밀번호 인증 정보 잠금
→ 비밀번호 기반 로그인 제한
```

실무에서는 계정을 바로 삭제하기보다 먼저 잠근 뒤 필요한 데이터와 소유 파일을 확인하는 방식이 더 안전할 수 있다.

## 계정 잠금 해제 - `passwd -u`

```bash
sudo passwd -u tempuser
```

`-u`는 unlock을 의미한다.

## 비밀번호 상태 확인 - `passwd -S`

```bash
sudo passwd -S tempuser
```

`-S`는 Status를 의미하며 사용자의 비밀번호 상태를 확인한다.

예:

```text
tempuser L 2026-09-01 0 99999 7 -1
```

두 번째 필드가 현재 상태를 나타낸다.

```text
P  → Password set, 비밀번호가 설정됨
L  → Locked, 비밀번호가 잠김
NP → No Password, 비밀번호가 설정되지 않음
```

현재 단계에서는 `P`, `L`, `NP`를 중심으로 이해한다.

뒤쪽 값들은 마지막 비밀번호 변경일, 최소/최대 사용 일수, 만료 경고 일수 등 비밀번호 정책과 관련된 값이다.

## 사용자 삭제 - `userdel`

사용자 계정 삭제:

```bash
sudo userdel tempuser
```

기본 `userdel`은 사용자 계정 정보를 삭제하지만 홈 디렉터리까지 자동으로 삭제하지 않는다.

```text
sudo userdel tempuser
→ 사용자 계정 삭제
→ /home/tempuser는 남을 수 있음
```

계정이 실제로 삭제됐는지 확인:

```bash
id tempuser
```

정상적으로 삭제됐다면 해당 사용자가 존재하지 않는다는 메시지가 나온다.

## 홈 디렉터리까지 같이 삭제 - `userdel -r`

```bash
sudo userdel -r tempuser
```

`-r`을 사용하면 사용자 계정과 함께 홈 디렉터리 및 일부 사용자 데이터를 제거한다.

```text
userdel
→ 계정 삭제, 홈 디렉터리는 남길 수 있음

userdel -r
→ 계정 + 홈 디렉터리까지 제거
```

운영 서버에서는 중요한 사용자 데이터가 있을 수 있으므로 `-r` 사용 전 반드시 확인과 백업이 필요하다.

## 실습

실습용 사용자 생성:

```bash
sudo adduser tempuser
```

계정 잠금:

```bash
sudo passwd -l tempuser
```

상태 확인:

```bash
sudo passwd -S tempuser
```

실습 결과에서 `L`을 확인하여 잠금 상태가 정상적으로 적용됐음을 확인했다.

잠금 해제:

```bash
sudo passwd -u tempuser
```

사용자 삭제:

```bash
sudo userdel tempuser
```

홈 디렉터리 확인:

```bash
ls -ld /home/tempuser
```

실습에서는 계정은 삭제됐지만 `/home/tempuser` 디렉터리는 남아 있는 것을 확인했다.

## 사용자가 삭제됐는데 소유자가 숫자로 보이는 이유

사용자 삭제 전에는 다음처럼 사용자 이름이 보일 수 있다.

```text
testuser testuser
```

계정을 삭제한 뒤 남아 있는 파일이나 홈 디렉터리를 확인하면 다음처럼 UID/GID 숫자가 표시될 수 있다.

```text
1001 1001
```

Linux 파일은 실제로 소유권을 사용자 이름 문자열이 아니라 UID/GID 값으로 관리한다.

```text
파일 소유권
→ UID 1001 / GID 1001 저장

사용자 계정 존재
→ 시스템이 1001을 사용자/그룹 이름으로 변환해서 표시

사용자 계정 삭제
→ UID 1001과 연결된 이름을 찾을 수 없음
→ 숫자 1001 그대로 표시
```

따라서 계정 삭제 후에도 그 사용자가 소유하던 파일이 남아 있을 수 있으며, 운영자는 이를 확인해야 한다.

## 안전한 계정 정리 흐름

바로 `userdel -r`을 실행하기보다 다음과 같은 순서로 접근한다.

```text
사용자 확인
→ 계정 잠금
→ 홈 디렉터리/데이터 확인
→ 필요한 데이터 백업
→ 소유 파일 확인
→ 사용자 삭제
→ 남은 파일/디렉터리 확인
→ 최종 검증
```

## Troubleshooting - userdel 후 홈 디렉터리가 남아 있음

### 상황

```bash
sudo userdel tempuser
```

을 실행했는데 `/home/tempuser`가 그대로 남아 있다.

초보 운영자는 사용자 삭제가 실패했다고 생각할 수 있다.

### 확인

계정이 실제로 삭제됐는지 확인한다.

```bash
id tempuser
```

홈 디렉터리가 남아 있는지도 확인한다.

```bash
ls -ld /home/tempuser
```

### 원인

기본 `userdel`은 홈 디렉터리까지 삭제하는 명령이 아니다.

따라서:

```text
계정 삭제 성공
+
홈 디렉터리 존재
```

상태가 동시에 발생할 수 있으며 정상 동작이다.

### 홈 디렉터리까지 같이 삭제하려면

```bash
sudo userdel -r tempuser
```

`-r` 옵션이 필요하다.

## 실습에서 얻은 핵심 포인트

```text
passwd -l
→ 비밀번호 잠금

passwd -u
→ 비밀번호 잠금 해제

passwd -S
→ 비밀번호 상태 확인

L
→ Locked

P
→ Password set

userdel
→ 사용자 계정 삭제

userdel -r
→ 사용자 계정 + 홈 디렉터리 삭제

사용자 삭제 후 1001 같은 숫자 표시
→ 파일에 남아 있는 UID/GID를 이름으로 변환할 계정 정보가 사라졌기 때문
```

## 운영 관점 핵심

사용자를 삭제해야 하는 상황에서도 바로 삭제 명령을 실행하는 것보다 먼저 계정을 잠그고, 데이터와 소유 파일을 확인한 뒤 필요한 경우 삭제하는 것이 안전하다.

```text
확인 → 잠금 → 데이터 점검/백업 → 삭제 → 검증
```
