# Linux Command & Option Reference — Day 0 ~ Day 5

> 지금까지 학습한 Linux 명령어를 **처음 보는 사람도 다시 이해할 수 있도록** 누적 정리한 Reference다. 단순 옵션 목록이 아니라 `개념 → 명령어 → 옵션 → 인자 → 출력 → 실무 사용 → 주의점` 순서로 확인한다.

## 📌 이 문서의 기준

```text
Linux Master 2급 수준 기본 개념 = 최소 누락 방지선
+
Linux 동작 원리
+
명령어 semantics
+
Option / Argument
+
출력 해석
+
실무 운영 흐름
+
Troubleshooting / 검증
```

처음 등장하는 기호와 숫자는 의미를 생략하지 않는다.

---

# 1. 명령어 문법

일반 구조:

```bash
command [option] [argument]
```

예:

```bash
ls -l /etc
```

```text
ls   → Command
-l   → Option: 동작/출력 방식을 변경
/etc → Argument: 처리 대상
```

### Short option

```bash
ls -l
ls -a
ls -al
```

결합 가능한 한 글자 옵션은 `-al`처럼 묶을 수 있다.

### Long option

```bash
ps --forest
```

보통 `--` 뒤에 긴 이름을 쓴다.

### `ps`는 예외적으로 여러 스타일을 지원

```text
ps -ef      → UNIX/POSIX style
ps aux      → BSD style
ps --forest → GNU long option
```

`aux`는 단일 단어 옵션이 아니라 `a`, `u`, `x`의 조합이다.

---

# 2. Shell / Path

## `pwd` — Print Working Directory

```bash
pwd
```

현재 작업 Directory의 절대경로 출력.

운영 안전 습관:

```text
pwd → ls → 대상 확인 → 변경 → 검증
```

## `cd` — Change Directory

```bash
cd /etc
cd ..
cd ~
```

```text
/  = Root Directory
.  = 현재 Directory
.. = 부모 Directory
~  = 현재 사용자 Home
```

```text
/etc    → 절대경로
etc     → 현재 위치 기준 상대경로
```

---

# 3. `ls` — List

```bash
ls
```

- `-l`: long format. Permission/Owner/Group/Size/mtime 등
- `-a`: all. 숨김 항목 포함
- `-d`: Directory 내부가 아니라 Directory 자체
- `-h`: human-readable size. 보통 `-lh`

```bash
ls -al
ls -ld /tmp
ls -lh file
```

`ls -l` Permission 예:

```text
-rw-r----- 1 linuxuser ops 1024 Sep 7 app.conf
```

```text
-           → regular file
rw-         → Owner
r--         → Group
---         → Others
linuxuser   → Owner name
ops         → Group name
```

---

# 4. 파일/Directory 조작

## `mkdir` — Make Directory

```bash
mkdir testdir
```

`.txt` 이름을 붙여도 Directory다.

## `touch`

```bash
touch file.txt
```

파일이 없으면 빈 파일 생성, 있으면 timestamp 갱신.

## `cp` — Copy

```bash
cp src dst
```

자주 볼 옵션:

```text
-r/-R → Directory 재귀 복사
-i    → overwrite 전 확인
-p    → metadata 보존 시도
```

## `mv` — Move

```bash
mv src dst
```

이동 또는 rename.

## `rm` — Remove

```bash
rm file
```

```text
-r → 재귀 삭제
-f → 강제 처리
-i → 삭제 전 확인
```

> `rm -rf`는 영향 범위가 크므로 경로 확인 없이 습관적으로 사용하지 않는다.

---

# 5. 파일 내용 확인

## `cat` — Concatenate

짧은 파일 전체 출력.

## `less`

긴 파일 탐색.

```text
/문자열 → 검색
n       → 다음 결과
q       → 종료
```

## `head`

```bash
head -n 10 file
```

`-n` = number of lines.

## `tail`

```bash
tail -n 20 file
tail -f app.log
```

- `-n`: 줄 수
- `-f`: follow, 새 줄을 계속 추적

## `history`

Shell 명령 이력 확인.

---

# 6. Standard Stream / File Descriptor

기본 Stream:

```text
stdin  = Standard Input  = FD 0
stdout = Standard Output = FD 1
stderr = Standard Error  = FD 2
```

File Descriptor는 Process가 열린 I/O 객체를 식별하는 작은 정수다.

## Redirection

```bash
command > out.log
```

stdout(FD1) overwrite.

```bash
command >> out.log
```

stdout append.

```bash
command 2> error.log
```

stderr(FD2)만 저장.

```bash
command > app.log 2>&1
```

```text
stdout → app.log
stderr → 현재 stdout 목적지(app.log)
```

## Pipe `|`

```bash
command1 | command2
```

command1 stdout → Pipe → command2 stdin.

---

# 7. 검색 — `find`, `grep`

## `find`

```bash
find [start-path] [condition]
```

```text
-name  → 이름 검색, 대소문자 구분
-iname → 이름 검색, 대소문자 무시
-type f → regular file
-type d → directory
-uid N → UID 기준
```

예:

```bash
find /var/log -type f -name "*.log"
```

## `grep`

내용에서 Pattern과 일치하는 줄 검색.

```text
-i → ignore case
-n → line number
-v → invert match
-r/-R → 재귀 검색
-c → 매칭 줄 수
-E → extended regex
```

```bash
grep -in "error" app.log
```

---

# 8. 집계 — `wc`, `sort`, `uniq`, `cut`, `awk`

## `wc`

Word Count.

```bash
wc -l file
```

`-l` = lines.

## `sort`

```text
-n → numeric
-r → reverse
```

```bash
sort -nr
```

숫자 큰 값부터.

## `uniq`

**인접 중복** 처리.

```bash
sort file | uniq -c
```

`-c` = count.

## `cut`

```bash
cut -d ':' -f 1 file
```

```text
-d → delimiter
-f → fields
```

## `awk`

Record/Field 기반 처리.

```text
$0 → 전체 Record
$1 → 첫 Field
$2 → 둘째 Field
NR → 현재 Record Number
```

```bash
awk '$4 >= 400 {print $1, $4}' access.log
```

---

# 9. `sed` — Stream Editor

기본 치환:

```bash
sed 's/old/new/' file
```

```text
s → substitute
g → 한 줄의 모든 match
```

```bash
sed 's/a/b/g' file
```

옵션/명령:

```text
-n → 기본 자동 출력 억제
p  → print
d  → current pattern space 삭제
-i → in-place, 원본 수정
```

```bash
sed -n '2,4p' file
sed -i 's/dev/prod/' file
```

운영에서는 `-i` 전 preview + backup을 우선한다.

---

# 10. `diff`

```bash
diff before after
diff -u before after
```

전통 형식:

```text
a → add
c → change
d → delete
```

`-u` = unified diff.

---

# 11. User / Group

## `whoami`

현재 effective username.

## `id`

```bash
id
id testuser
```

UID, Primary GID, Supplementary Groups.

## `groups`

사용자 Group 목록.

## `getent`

NSS(Name Service Switch) 경로로 DB 조회.

```bash
getent passwd testuser
getent group ops
```

`grep /etc/passwd`보다 LDAP 등 외부 Identity Source 환경에서도 범용적이다.

---

# 12. 사용자/Group 관리

## `adduser`

Ubuntu/Debian higher-level 사용자 생성 도구.

## `useradd`

```bash
useradd -m testuser
```

`-m` = Home 생성.

## `groupadd`

Group 생성.

## `usermod`

```bash
usermod -aG ops testuser
```

```text
-G → supplementary group list
-a → append
```

추가 목적이면 `-aG`를 함께 쓴다.

---

# 13. `su`, `sudo`

## `su`

```bash
su testuser
su - testuser
```

`-`는 target user의 login environment를 더 가깝게 구성한다.

## `sudo`

sudoers 정책에 따라 다른 사용자(보통 root) 권한으로 명령 실행.

```bash
sudo -l
```

`-l` = list allowed commands.

---

# 14. 계정 잠금/삭제

## `passwd`

```text
-l → lock password authentication
-u → unlock
-S → status
```

대표 상태:

```text
P  → password set
L  → locked
NP → no password
```

Password lock이 SSH public key 등 모든 인증 경로 차단을 뜻하지는 않는다.

## `userdel`

```bash
userdel user
userdel -r user
```

`-r` = Home/mail spool 등 관련 데이터 제거.

Home 밖 사용자 소유 파일은 별도로 남을 수 있다.

---

# 15. Permission — `rwx`, `chmod`

## `r=4`, `w=2`, `x=1`

이 숫자는 3-bit binary 자리값이다.

```text
r w x
4 2 1
```

```text
--- = 000 = 0
--x = 001 = 1
-w- = 010 = 2
-wx = 011 = 3
r-- = 100 = 4
r-x = 101 = 5
rw- = 110 = 6
rwx = 111 = 7
```

3자리는:

```text
Owner | Group | Others
```

예:

```text
640 = rw-r-----
755 = rwxr-xr-x
```

## `chmod`

```bash
chmod 640 file
```

문자 방식:

```text
u = Owner
g = Group
o = Others
a = All
+ = add
- = remove
= = exact set
```

```bash
chmod u+x file
chmod g-w dir
chmod g=rx dir
```

---

# 16. 파일 vs Directory `rwx`

| Permission | File | Directory |
|---|---|---|
| `r` | 내용 읽기 | Entry 이름 목록 조회 |
| `w` | 내용 수정 | Entry 생성/삭제/rename |
| `x` | 실행 | traverse/search |

파일이 readable이어도 부모 Directory `x`가 없으면 접근이 실패할 수 있다.

파일 삭제는 파일 자체 `w`보다 부모 Directory `w+x`와 밀접하다.

---

# 17. Ownership — `chown`, `chgrp`

```bash
chown user file
chown user:group file
chgrp group file
```

```text
chmod → Permission
chown → Owner/Group ownership
chgrp → Group ownership
```

`-R`은 재귀 적용이므로 영향 범위에 주의한다.

---

# 18. Special Permission

앞자리 특수 bit:

```text
SetUID = 4
SetGID = 2
Sticky = 1
```

```text
4755 → SetUID + 755
2770 → SetGID + 770
1777 → Sticky + 777
```

### SetUID

Executable의 Effective UID를 file Owner 기준으로 사용할 수 있게 한다.

대표: `/usr/bin/passwd`.

### SetGID

Directory에서는 새 항목이 부모 Directory Group을 상속하게 한다.

### Sticky Bit

공용 writable Directory에서 다른 사용자의 파일 delete/rename을 제한한다.

대표: `/tmp`.

```text
s/t → 특수 bit + execute 있음
S/T → 특수 bit는 있으나 execute 없음
```

---

# 19. Process — `ps`, `pgrep`, `top`, `kill`

## 핵심 개념

```text
Program → 저장된 실행 코드
Process → Kernel이 관리하는 실행 단위
PID     → Process ID
PPID    → Parent PID
```

### State 기초

```text
R = Running/Runnable
S = Interruptible Sleep
D = Uninterruptible Sleep
T = Stopped
Z = Zombie
```

## `ps`

```bash
ps -ef
```

```text
-e → all processes
-f → full format
```

```bash
ps aux
```

```text
a → selection 확대
u → user-oriented format
x → no-TTY processes 포함
```

```bash
ps -fp PID
```

```text
-f → full
-p → PID 선택
```

## `pgrep`

```bash
pgrep -a sleep
```

`-a` = PID + command line.

```bash
pgrep -af "sleep 900"
```

`-f` = full command line을 match 대상으로 사용.

## `top`

실시간 Process/System monitor. `q` 종료.

## `kill`

Signal 전송 명령.

```text
kill PID     → 기본 SIGTERM(15)
kill -15 PID → SIGTERM
kill -9 PID  → SIGKILL
```

운영:

```text
대상 확인 → TERM → 검증 → 필요할 때만 KILL
```

---

# 20. Job Control / `nohup`

```text
Process → Kernel의 시스템 전체 실행 단위
Job     → 현재 Shell의 작업 제어 단위
```

```bash
jobs -l
```

```text
-l → Job ID + PID
-r → Running Job
-s → Stopped Job
```

```bash
bg %1
fg %1
```

`%1` = Job Specifier, Job 1.

```text
Ctrl+C → 보통 SIGINT
Ctrl+Z → 보통 SIGTSTP
```

### `&`

처음부터 background 실행하는 Shell operator.

### `nohup`

SIGHUP을 무시하도록 command 실행.

```bash
nohup command > app.log 2>&1 &
```

```text
nohup → SIGHUP 처리
>      → stdout redirect
2>&1   → stderr를 stdout 목적지로
&      → background
```

운영 Service는 보통 systemd 같은 Service Manager가 더 적합하다.

---

# 21. 누적 운영 체크리스트

## 파일/설정 변경

```text
whoami / hostname
→ pwd
→ ls -l / ls -ld
→ 내용 확인
→ backup
→ preview
→ 변경
→ diff/grep
→ 서비스/기능 검증
```

## Permission 문제

```text
id
→ ls -l file
→ ls -ld parent
→ Owner/Group 확인
→ 필요한 rwx 판단
→ 최소 변경
→ 실제 접근 검증
```

## Process 문제

```text
pgrep -a
→ ps -fp PID
→ top
→ 영향 판단
→ SIGTERM
→ 검증
→ 필요 시 SIGKILL
```

---

# 22. 문서화 기준

앞으로 새 개념은 다음을 생략하지 않는다.

```text
1. 처음 보는 사람을 위한 한 줄 정의
2. 왜 필요한지
3. 내부적으로 무엇을 의미하는지
4. 기호/숫자/약어의 의미
5. 명령어 문법
6. Option 각각의 의미
7. Argument 의미
8. 중요한 출력 Column 해석
9. 실제 Ubuntu 실습
10. 실무 사용 사례
11. 위험 요소
12. Troubleshooting
13. 검증 방법
```

목표는 자격증 암기 노트가 아니라 **주니어 Linux/System/Infra 엔지니어가 면접과 실제 장애 상황에서 이유를 설명할 수 있는 수준**이다.
