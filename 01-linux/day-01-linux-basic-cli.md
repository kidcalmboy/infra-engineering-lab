# Day 1 - Linux 기본 CLI와 파일시스템

> Linux 서버에서 현재 위치와 대상을 확인하고, 파일·디렉터리를 안전하게 조작하며, 출력·리다이렉션·파이프와 핵심 디렉터리 구조를 학습했다.

## 📌 이번에 배운 내용

- `pwd`, `ls`, `cd`, `mkdir`, `touch`, `cp`, `mv`, `rm`
- 절대경로와 상대경로, `.`, `..`, `~`
- `cat`, `less`, `head`, `tail`, `tail -f`, `history`
- `>`, `>>` 리다이렉션 차이
- 파이프 `|`
- `/etc`, `/var/log`, `/home`, `/tmp`, `/proc`, `/dev`, `/usr`
- 잘못된 작업 디렉터리에서 명령을 실행한 실수와 해결 과정
- 설정 파일 수정 전 확인·백업 습관

## 📚 목차

1. 서버 작업 전 상태 확인
2. 파일과 디렉터리 조작
3. 경로 개념
4. 파일 내용 확인
5. 리다이렉션과 파이프
6. Linux 핵심 디렉터리
7. 실습
8. 헷갈리기 쉬운 부분
9. Troubleshooting
10. 실무 포인트
11. 핵심 정리
12. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 의미 / 용도 |
|---|---|
| `pwd` | Print Working Directory, 현재 작업 디렉터리 확인 |
| `ls` | 현재 위치의 파일·디렉터리 목록 확인 |
| `ls -l` | 상세 목록 확인 |
| `ls -a` | 숨김 파일 포함 목록 확인 |
| `cd path` | 디렉터리 이동 |
| `mkdir dir` | 디렉터리 생성 |
| `touch file` | 빈 파일 생성 또는 타임스탬프 갱신 |
| `cp src dst` | 파일 복사 |
| `mv src dst` | 파일 이동 또는 이름 변경 |
| `rm file` | 파일 삭제 |
| `cat file` | 짧은 파일 전체 출력 |
| `less file` | 긴 파일을 페이지 단위로 확인 |
| `head -n 10 file` | 앞부분 확인 |
| `tail -n 10 file` | 뒷부분 확인 |
| `tail -f file` | 새로 추가되는 내용을 실시간 확인 |
| `history` | 이전 명령 기록 확인 |

---

## 1. 서버 작업 전 상태 확인

### 한 줄 정의

Linux 운영에서는 명령을 실행하기 전에 **누구로 로그인했고, 어느 서버에서, 어느 위치에서 작업 중인지** 확인하는 습관이 중요하다.

```bash
whoami
hostname
pwd
ip addr
```

| 명령어 | 확인 내용 |
|---|---|
| `whoami` | 현재 실행 사용자 |
| `hostname` | 현재 서버 이름 |
| `pwd` | 현재 작업 디렉터리 |
| `ip addr` | 네트워크 인터페이스와 IP 주소 |

### 실무에서 중요한 이유

같은 명령도 사용자, 서버, 작업 디렉터리가 다르면 완전히 다른 결과를 만들 수 있다. 특히 삭제·이동·설정 변경 전에는 작업 대상을 확인해야 한다.

---

## 2. 파일과 디렉터리 조작

### 디렉터리 생성

```bash
mkdir linux-test
```

`mkdir` = **make directory**

### 파일 생성

```bash
touch test1.txt
```

학습에서는 빈 실습 파일을 빠르게 만들기 위해 사용했다.

### 복사

```bash
cp test1.txt backup/
```

`cp` = **copy**

원본을 남기고 복사본을 만든다.

### 이동·이름 변경

```bash
mv test2.txt config.txt
```

`mv` = **move**

같은 디렉터리에서 대상 이름만 바꾸면 이름 변경처럼 동작한다.

### 삭제

```bash
rm test1.txt
```

`rm` = **remove**

삭제는 되돌리기 어려울 수 있으므로 실행 전 현재 위치와 대상을 반드시 확인한다.

```text
pwd
→ ls
→ 대상 확인
→ 삭제
→ 다시 ls로 검증
```

---

## 3. 경로 개념

### 절대경로

루트 디렉터리 `/`부터 시작하는 전체 경로다.

```text
/home/linuxuser/test.txt
```

현재 위치가 어디든 같은 대상을 가리킨다.

### 상대경로

현재 작업 디렉터리를 기준으로 해석한다.

```text
backup/test1.txt
```

현재 위치가 바뀌면 의미도 달라질 수 있다.

### 특수 경로

| 표현 | 의미 |
|---|---|
| `.` | 현재 디렉터리 |
| `..` | 상위 디렉터리 |
| `~` | 현재 사용자의 홈 디렉터리 |
| `/` | 파일시스템 최상위 루트 |

예:

```bash
cd ..
cd ~
cd /
```

---

## 4. 파일 내용 확인

### `cat`

```bash
cat server.log
```

짧은 파일을 한 번에 확인할 때 적합하다.

### `less`

```bash
less server.log
```

긴 파일을 스크롤하며 확인할 때 사용한다.

주요 조작:

```text
/ERROR → ERROR 검색
n      → 다음 검색 결과
q      → 종료
```

### `head`, `tail`

```bash
head -n 5 server.log
tail -n 5 server.log
```

로그의 앞부분 또는 최신 부분을 빠르게 확인할 때 사용한다.

### `tail -f`

```bash
tail -f server.log
```

파일 끝에 새 내용이 추가되는 것을 실시간으로 확인한다.

중지:

```text
Ctrl + C
```

### `history`

```bash
history
```

이전에 실행한 명령을 확인한다. 반복 작업이나 직전에 무엇을 실행했는지 점검할 때 유용하다.

---

## 5. 리다이렉션과 파이프

### `>` - 덮어쓰기

```bash
echo "hello" > test.txt
```

명령 출력을 파일에 기록하며 기존 파일 내용이 있다면 덮어쓴다.

### `>>` - 추가

```bash
echo "world" >> test.txt
```

기존 내용을 유지하고 파일 끝에 추가한다.

### 비교

| 기호 | 의미 |
|---|---|
| `>` | 출력 → 파일, 기존 내용 덮어쓰기 |
| `>>` | 출력 → 파일, 기존 내용 뒤에 추가 |
| `|` | 왼쪽 명령 출력 → 오른쪽 명령 입력 |

### 파이프 `|`

```bash
history | tail -n 5
```

왼쪽 명령 결과를 오른쪽 명령이 다시 처리한다.

```text
history 출력
→ tail 입력
→ 마지막 5줄 출력
```

---

## 6. Linux 핵심 디렉터리

Linux는 Windows의 드라이브 문자처럼 분리된 구조가 아니라 `/` 아래에 모든 경로가 연결된다.

| 경로 | 역할 | 운영 관점 |
|---|---|---|
| `/` | 최상위 루트 | 모든 절대경로의 시작 |
| `/etc` | 시스템·서비스 설정 | 설정 오류 확인 |
| `/var` | 계속 변하는 데이터 | 로그·캐시·서비스 데이터 |
| `/var/log` | 로그 | 장애 원인 분석 |
| `/home` | 일반 사용자 홈 | 사용자 작업 공간 |
| `/tmp` | 임시 파일 | 단기 테스트 파일 |
| `/proc` | 가상 파일시스템 | 현재 커널·프로세스·메모리 상태 |
| `/dev` | 장치 파일 | 디스크·터미널 등 장치 표현 |
| `/usr` | 프로그램·라이브러리·공유 데이터 | 설치된 소프트웨어 구성 요소 |

### `/proc`는 로그 디렉터리가 아니다

```bash
head /proc/meminfo
```

`/proc/meminfo`는 디스크에 저장된 과거 로그가 아니라 커널이 현재 시스템 상태를 보여주는 가상 파일이다.

---

## 7. 🧪 실습

### 파일·디렉터리 조작

```bash
cd ~
mkdir linux-test
cd linux-test
touch test1.txt test2.txt
mkdir backup
cp test1.txt backup/
mv test2.txt config.txt
pwd
ls
ls backup
```

구조:

```text
~/linux-test/
├── backup/
│   └── test1.txt
├── config.txt
└── test1.txt
```

### 로그 파일 만들기

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

### 핵심 디렉터리 확인

```bash
cd /
ls
cat /etc/hostname
head /proc/meminfo
```

---

## 8. ⚠️ 헷갈리기 쉬운 부분

### 명령어가 맞으면 결과도 항상 원하는 대로 나오는가?

아니다.

> ⚠️ 처음에는 `touch test1.txt` 같은 명령이 정확하면 원하는 위치에 파일이 만들어질 것으로 생각하기 쉽지만, **현재 작업 디렉터리**가 다르면 올바른 명령으로도 잘못된 위치에 파일을 만들 수 있다.

그래서 `pwd`가 중요하다.

### `cp`와 `mv`

```text
cp → 원본 유지 + 복사본 생성
mv → 원본의 위치 또는 이름 변경
```

### `>`와 `>>`

`>`를 설정 파일에 잘못 사용하면 기존 내용이 사라질 수 있다.

```bash
echo "test" > nginx.conf
```

이 명령은 기존 `nginx.conf` 내용을 보존하지 않는다.

---

## 9. 🔧 Troubleshooting

### 문제 1 - 파일이 예상한 디렉터리에 없음

처음에는 `linux-test` 디렉터리를 만든 뒤 이동하지 않고 파일을 생성해 홈 디렉터리에 파일이 생겼다.

### 원인

현재 작업 디렉터리를 확인하지 않았다.

### 확인

```bash
pwd
ls
```

### 해결

잘못 만든 파일을 확인하고 올바른 디렉터리에서 다시 생성했다.

```bash
rm test1.txt test2.txt
cd linux-test
touch test1.txt test2.txt
ls
```

### 배운 점

**정확한 명령 + 잘못된 작업 위치 = 잘못된 결과**가 될 수 있다.

### 문제 2 - 설정 파일 덮어쓰기 위험

```bash
echo "test" > nginx.conf
```

`>`는 overwrite이므로 기존 설정을 잃을 수 있다.

안전한 흐름:

```bash
cat nginx.conf
cp nginx.conf nginx.conf.bak
```

이후 수정하고 결과를 검증한다.

---

## 10. 💼 실무 포인트

파일 작업은 다음 순서를 기본 습관으로 만든다.

```text
현재 사용자 확인
→ 서버 확인
→ 현재 위치 확인
→ 대상 확인
→ 필요 시 백업
→ 명령 실행
→ 결과 검증
```

서비스 장애에서는 특히:

```text
상태 확인
→ /var/log 로그 확인
→ /etc 설정 확인
→ 수정
→ 서비스 검증
```

흐름을 자주 사용하게 된다.

---

## 11. ✅ 핵심 정리

- `pwd`는 현재 작업 디렉터리를 확인한다.
- 절대경로는 `/`부터 시작하고 상대경로는 현재 위치를 기준으로 한다.
- `cp`는 복사, `mv`는 이동·이름 변경, `rm`은 삭제다.
- `cat`은 짧은 파일, `less`는 긴 파일, `tail -f`는 실시간 로그 확인에 유용하다.
- `>`는 덮어쓰기, `>>`는 추가다.
- `|`는 명령어의 출력을 다른 명령어의 입력으로 전달한다.
- `/etc`는 설정, `/var/log`는 로그, `/home`은 사용자 홈, `/proc`는 현재 시스템 정보를 제공하는 가상 파일시스템이다.
- 파일 변경 전에는 **위치·대상·내용을 확인하고 필요하면 백업**한다.

---

## 12. 🧠 복습 문제

1. 절대경로와 상대경로의 차이는 무엇인가?
2. `cd ..`와 `cd ~`는 각각 어디로 이동하는가?
3. `cp`와 `mv`의 차이는 무엇인가?
4. `>`와 `>>`의 차이는 무엇인가?
5. `tail -f`는 어떤 상황에서 사용하는가?
6. `/etc`와 `/var/log`는 각각 어떤 역할을 하는가?
7. 올바른 명령을 실행했는데 파일이 엉뚱한 위치에 생성될 수 있는 이유는 무엇인가?
