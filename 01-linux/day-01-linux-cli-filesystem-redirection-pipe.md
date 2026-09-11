# Day 1 — Linux 기본 CLI와 파일시스템

> Linux 서버에서 **현재 위치·대상·출력 흐름을 정확히 이해하고 안전하게 파일을 조작하는 방법**을 학습했다. 명령어 암기보다 `경로 → 파일/디렉터리 → 표준 입출력 → 리다이렉션 → 파이프 → 핵심 디렉터리`가 어떻게 연결되는지 이해하는 것이 목표다.

## 📌 이번에 배운 내용

- Shell, Prompt, Command, Option, Argument
- `pwd`, `ls`, `cd`, `mkdir`, `touch`, `cp`, `mv`, `rm`
- 절대경로 / 상대경로 / `/` / `.` / `..` / `~`
- 일반 파일과 디렉터리 차이
- `cat`, `less`, `head`, `tail`, `history`
- `head -n`, `tail -n`, `tail -f`
- Standard Input / Output / Error의 기초
- `>`, `>>`, `|`
- `/etc`, `/var`, `/var/log`, `/home`, `/tmp`, `/proc`, `/dev`, `/usr`
- 작업 전 `pwd → ls → 확인 → 실행 → 검증` 절차

## 📚 목차

1. Shell과 명령어 구조
2. Linux 파일시스템과 경로
3. 파일과 디렉터리 생성·조작
4. 파일 내용 확인
5. 표준 입출력
6. Redirection과 Pipe
7. 주요 시스템 디렉터리
8. 주요 명령어와 옵션
9. 실제 실습
10. 헷갈리기 쉬운 부분
11. Troubleshooting
12. 실무 포인트
13. 핵심 정리
14. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 옵션/인자 | 의미 | 실무 사용 |
|---|---|---|---|
| `pwd` | 없음 | 현재 작업 디렉터리의 절대경로 출력 | 삭제·수정 전 위치 확인 |
| `ls -al` | `-a` all, `-l` long | 숨김 파일 포함 상세 목록 | 권한/소유자/파일 확인 |
| `ls -ld dir` | `-d` directory itself | 디렉터리 내부가 아니라 자체 정보 | 디렉터리 권한 확인 |
| `cd /etc` | `/etc` = argument | 작업 디렉터리 변경 | 설정 디렉터리 이동 |
| `mkdir dir` | `dir` = 새 디렉터리명 | 디렉터리 생성 | 작업 공간 생성 |
| `touch file` | `file` = 대상 | 빈 파일 생성 또는 timestamp 갱신 | 실습 파일 생성 |
| `cp src dst` | source / destination | 복사 | 설정 백업 |
| `mv src dst` | source / destination | 이동 또는 rename | 파일 배치/이름 변경 |
| `rm file` | 대상 파일 | 삭제 | 불필요 파일 제거 |
| `head -n 10 file` | `-n` number | 앞 10줄 확인 | 로그 앞부분 확인 |
| `tail -n 20 file` | `-n` number | 끝 20줄 확인 | 최신 로그 확인 |
| `tail -f file` | `-f` follow | 파일에 추가되는 줄 지속 확인 | 실시간 로그 관찰 |

---

## 1. Shell과 명령어 구조

### Shell이란?

Shell은 사용자의 명령을 해석해서 프로그램을 실행하고 결과를 보여주는 인터페이스다. 현재 Ubuntu Server에서는 Bash를 주로 사용한다.

```text
사용자 입력
→ Shell(Bash)
→ 명령 해석
→ 프로그램 실행 요청
→ Kernel
→ 결과 출력
```

### Prompt란?

예:

```text
linuxuser@ubuntu-server:~$
```

대략 다음 정보를 보여준다.

```text
linuxuser     → 현재 사용자
ubuntu-server → hostname
~             → 현재 위치(홈 디렉터리)
$             → 일반 사용자 shell prompt
```

root 계정에서는 흔히 `#`가 보일 수 있지만, prompt 문자만으로 권한을 단정하지 말고 `whoami`, `id`로 확인하는 습관이 좋다.

### Command / Option / Argument

예:

```bash
ls -l /etc
```

```text
ls   → command
-l   → option
/etc → argument
```

**Option**은 명령의 동작 방식을 바꾸고, **Argument**는 명령이 처리할 대상이나 값을 지정한다.

---

## 2. Linux 파일시스템과 경로

Linux의 파일시스템은 최상위 `/`에서 시작하는 하나의 트리 구조로 보인다.

```text
/
├─ etc
├─ home
├─ usr
├─ var
└─ tmp
```

Windows의 `C:`, `D:`처럼 사용자에게 드라이브 문자가 직접 보이는 방식과 다르다. 다른 디스크도 특정 디렉터리에 **mount**되어 하나의 트리에 연결될 수 있다.

### 절대경로

`/`부터 시작하는 전체 경로다.

```text
/home/linuxuser/test.txt
```

현재 위치와 관계없이 같은 대상을 가리킨다.

### 상대경로

현재 작업 디렉터리를 기준으로 해석한다.

```text
backup/test.txt
```

현재 위치가 `/home/linuxuser`라면:

```text
/home/linuxuser/backup/test.txt
```

로 해석된다.

### 특수 경로 표현

| 표현 | 의미 |
|---|---|
| `/` | 파일시스템 최상위 Root Directory |
| `.` | 현재 디렉터리 |
| `..` | 상위 디렉터리 |
| `~` | 현재 사용자의 Home Directory |

예:

```bash
cd ~
cd ..
cd ./backup
```

### 왜 `pwd`가 중요한가?

상대경로를 사용하는 명령은 현재 위치에 따라 결과가 달라진다.

```bash
touch test.txt
```

이 명령 자체가 정확해도 현재 위치가 잘못되어 있으면 잘못된 디렉터리에 파일이 만들어진다.

---

## 3. 파일과 디렉터리 생성·조작

### `mkdir` — Make Directory

```bash
mkdir backup
```

디렉터리를 만든다.

중요:

```bash
mkdir test.txt
```

라고 해도 **`test.txt`라는 디렉터리**가 만들어진다. Linux에서 `.txt` 같은 확장자는 파일 형식을 강제로 결정하지 않는다.

### `touch`

```bash
touch test.txt
```

파일이 없으면 빈 일반 파일을 만들 수 있고, 이미 있으면 timestamp를 갱신한다.

### `cp` — Copy

```bash
cp service.conf service.conf.bak
```

원본을 유지하고 사본을 만든다.

실무에서는 설정 파일 변경 전 백업에 자주 사용한다.

### `mv` — Move

```bash
mv old.txt new.txt
```

위치 이동과 이름 변경에 사용한다. 같은 파일시스템 안에서는 rename에 가까운 동작으로 처리될 수 있다.

### `rm` — Remove

```bash
rm test.txt
```

파일을 삭제한다. 일반적으로 GUI 휴지통처럼 복구를 전제로 하지 않기 때문에 운영에서는 대상 확인이 중요하다.

기본 안전 흐름:

```text
pwd
→ ls
→ 대상 확인
→ rm
→ ls로 검증
```

---

## 4. 파일 내용 확인

### `cat` — Concatenate

```bash
cat app.conf
```

짧은 파일 전체를 한 번에 볼 때 적합하다. 원래 여러 파일을 이어 출력하는 기능도 있지만 실무에서는 짧은 파일 확인에 자주 쓴다.

### `less`

```bash
less /var/log/syslog
```

긴 파일을 페이지 단위로 탐색한다.

주요 조작:

```text
/ERROR → ERROR 검색
n      → 다음 검색 결과
q      → 종료
```

### `head`

```bash
head -n 10 file
```

`-n`은 **number of lines**를 지정한다. 앞 10줄을 본다.

### `tail`

```bash
tail -n 20 app.log
```

파일 끝 20줄을 본다.

### `tail -f`

```bash
tail -f app.log
```

`-f`는 **follow**다. 파일 끝에 새 줄이 추가되면 계속 따라가며 출력한다.

서비스 시작 직후 로그가 어떻게 변하는지 관찰할 때 매우 자주 쓴다.

### `history`

```bash
history
```

현재 Shell의 이전 명령 기록을 확인한다. 직전에 어떤 조작을 했는지 추적할 때도 유용하지만, 보안상 민감한 값을 명령행에 직접 넣으면 history에 남을 수 있다는 점도 알아둘 필요가 있다.

---

## 5. Standard Input / Output / Error

Linux 명령어는 기본적으로 세 개의 표준 스트림과 연결된다.

```text
stdin  → Standard Input  → FD 0
stdout → Standard Output → FD 1
stderr → Standard Error  → FD 2
```

### FD란?

FD(File Descriptor)는 Process가 열린 파일이나 입출력 스트림을 식별하기 위해 사용하는 작은 정수 번호다.

초심자 단계에서는 우선 다음만 정확히 기억하면 된다.

```text
0 = 입력
1 = 정상 출력
2 = 오류 출력
```

예를 들어 `cat`은 파일 내용을 읽어 stdout으로 보내고, 오류가 발생하면 오류 메시지를 stderr로 보낼 수 있다.

---

## 6. Redirection과 Pipe

### `>` — stdout 덮어쓰기

```bash
echo hello > file.txt
```

stdout(FD 1)의 목적지를 터미널 대신 파일로 바꾸고 기존 내용을 덮어쓴다.

### `>>` — stdout 추가

```bash
echo world >> file.txt
```

기존 내용을 유지하고 끝에 추가한다.

### `2>` — stderr 리다이렉션

```bash
command 2> error.log
```

오류 출력 FD 2만 파일로 보낸다.

### `2>&1`

```bash
command > output.log 2>&1
```

먼저 stdout을 `output.log`로 보내고, stderr(FD 2)를 **현재 stdout(FD 1)이 가는 곳과 같은 곳**으로 보낸다.

이 표현은 이후 `nohup`과 로그 처리에서도 다시 사용한다.

### Pipe `|`

```bash
history | tail -n 5
```

왼쪽 Process의 stdout을 오른쪽 Process의 stdin에 연결한다.

```text
history stdout
     ↓
    pipe
     ↓
tail stdin
```

즉 파이프는 단순히 화면 결과를 붙이는 것이 아니라 **프로세스 간 데이터 흐름**을 만든다.

---

## 7. 주요 시스템 디렉터리

| 경로 | 핵심 역할 | 운영 관점 |
|---|---|---|
| `/etc` | 시스템/서비스 설정 | 설정 오류 점검 |
| `/var` | 계속 변하는 데이터 | 로그·캐시·서비스 데이터 |
| `/var/log` | 전통적인 로그 파일 위치 | 장애 원인 분석 |
| `/home` | 일반 사용자 홈 | 사용자 작업 공간 |
| `/tmp` | 임시 데이터 | 테스트/임시 파일, 권한 주의 |
| `/proc` | Kernel/Process 정보를 보여주는 가상 FS | Process/Memory 상태 확인 |
| `/dev` | Device를 파일처럼 표현 | Disk/Terminal/Device 접근 |
| `/usr` | 프로그램·라이브러리·공유 데이터 | 설치된 소프트웨어 구성 |

### `/proc`는 일반 디스크 파일인가?

아니다. `/proc`는 **가상 파일시스템**으로 Kernel이 현재 시스템 상태를 파일 형태의 인터페이스로 보여준다.

```bash
head /proc/meminfo
```

이 출력은 과거 로그가 아니라 현재 메모리 관련 Kernel 정보를 보여준다.

### `/dev`가 왜 파일인가?

Unix/Linux는 많은 장치를 파일과 비슷한 인터페이스로 다룬다. `/dev/null`, Disk Device, Terminal Device 등이 여기에 나타난다.

---

## 8. 주요 명령어와 옵션

### `ls`

- `-l`: long listing. 권한, 링크 수, Owner, Group, Size, 수정시각 등 표시
- `-a`: all. `.`으로 시작하는 숨김 항목 포함
- `-d`: directory 자체를 표시
- `-h`: human-readable. 크기를 KB/MB/GB 식으로 보기 쉽게 표시. 보통 `-lh`로 사용

```bash
ls -al
ls -lh
ls -ld /tmp
```

### `cp`

앞으로 자주 볼 옵션:

- `-r` / `-R`: directory를 재귀적으로 복사
- `-i`: overwrite 전에 확인
- `-p`: mode/ownership/timestamps 등 보존 시도

현재 단계에서는 기본 복사를 먼저 정확히 이해하고, 운영에서 디렉터리 복사나 metadata 보존이 필요할 때 옵션을 선택한다.

### `rm`

자주 보게 될 옵션:

- `-r`: directory를 재귀적으로 삭제
- `-f`: 확인 없이 강제 처리
- `-i`: 삭제 전 확인

> `rm -rf`는 매우 강한 명령이므로 경로를 이해하지 못한 상태에서 습관적으로 사용하면 안 된다.

---

## 9. 🧪 실제 실습

```bash
cd ~
mkdir linux-test
cd linux-test
touch test1.txt test2.txt
mkdir backup
cp test1.txt backup/
mv test2.txt config.txt
pwd
ls -al
```

로그 실습:

```bash
echo "INFO server started" > server.log
echo "INFO user connected" >> server.log
echo "WARNING disk usage 80%" >> server.log
echo "ERROR database connection failed" >> server.log
```

확인:

```bash
cat server.log
head -n 2 server.log
tail -n 2 server.log
less server.log
```

---

## 10. ⚠️ 헷갈리기 쉬운 부분

> ⚠️ 파일 이름 확장자가 파일 종류를 결정하지 않는다. `mkdir a.txt`는 디렉터리다.

> ⚠️ `>`와 `>>`는 다르다. `>`는 기존 내용을 덮어쓸 수 있다.

> ⚠️ `|`는 파일 저장 기능이 아니다. 왼쪽 Process의 stdout을 오른쪽 Process의 stdin으로 연결한다.

> ⚠️ 정확한 명령어라도 현재 디렉터리가 틀리면 잘못된 대상을 변경할 수 있다.

---

## 11. 🔧 Troubleshooting

### 증상: 파일이 예상 위치에 없음

확인:

```bash
pwd
ls -al
```

원인:

```text
명령은 맞았지만 현재 작업 디렉터리를 잘못 이해함
```

조치:

```text
현재 위치 확인
→ 잘못 생성된 대상 확인
→ 필요한 경우 안전하게 제거/이동
→ 올바른 위치로 cd
→ 다시 실행
→ 검증
```

### 증상: 설정 파일 내용이 사라짐

가능한 원인:

```bash
echo value > config.conf
```

`>`가 기존 내용을 overwrite했을 수 있다.

예방:

```text
cat/less로 현재 내용 확인
→ cp로 백업
→ 수정
→ diff/grep으로 검증
```

---

## 12. 💼 실무 포인트

파일 작업은 다음 습관으로 연결한다.

```text
whoami / hostname
→ pwd
→ ls -l / ls -ld
→ 대상 내용 확인
→ 필요 시 백업
→ 명령 실행
→ 결과 검증
```

이후 서버 운영에서 설정 파일, 로그, 배포 파일, 백업 파일을 다룰 때 이 기본기가 그대로 사용된다.

---

## 13. ✅ 핵심 정리

- Shell은 명령을 해석하고 Program 실행을 요청한다.
- Command / Option / Argument는 서로 역할이 다르다.
- 절대경로는 `/`부터, 상대경로는 현재 작업 디렉터리부터 해석한다.
- `mkdir`는 directory, `touch`는 일반 파일 생성에 자주 사용한다.
- `cp`는 복사, `mv`는 이동/rename, `rm`은 삭제다.
- stdin=0, stdout=1, stderr=2다.
- `>`는 stdout overwrite, `>>`는 append, `|`는 프로세스 간 데이터 연결이다.
- `/etc`는 설정, `/var/log`는 로그, `/proc`는 Kernel/Process 정보를 제공하는 가상 파일시스템이다.

---

## 14. 🧠 복습 문제

1. Command, Option, Argument의 차이를 예시로 설명해보라.
2. 절대경로와 상대경로는 무엇이 다른가?
3. `mkdir test.txt`는 어떤 타입을 만드는가?
4. stdin/stdout/stderr와 FD 0/1/2를 연결해보라.
5. `>`와 `>>`의 차이는 무엇인가?
6. `command > app.log 2>&1`을 해석해보라.
7. Pipe `|`는 내부적으로 어떤 두 스트림을 연결하는가?
8. `/proc`와 `/var/log`는 어떤 점이 다른가?
