# Day 1 - Linux 기본 명령어와 파일·디렉터리 조작

## 학습 목표

- 현재 사용자, 서버 이름, 작업 위치, IP 주소를 직접 확인한다.
- 파일과 디렉터리를 만들고 복사·이동·삭제한다.
- 파일 내용을 전체/일부/실시간으로 확인하는 방법을 익힌다.
- `>`와 `>>`의 차이를 이해하고 파일 덮어쓰기 사고를 예방한다.
- 명령 실행 전 현재 위치와 대상을 확인하는 습관을 만든다.

## 실습 환경

```text
Windows Host
  └─ VirtualBox
      └─ Ubuntu Server Guest
          └─ OpenSSH Server (Port 22)
```

VirtualBox와 Ubuntu Server 설치, NAT 포트 포워딩, SSH 접속 과정은 [Day 0 실습 환경 구축 기록](day-00-virtualbox-ubuntu-server-setup.md)에 정리했다. 이번 실습은 Windows PowerShell에서 Ubuntu Server에 SSH로 접속한 상태에서 진행했다.

## 서버 기본 상태 확인

SSH 접속 후 다음 명령으로 현재 환경을 확인했다.

```bash
whoami
hostname
pwd
ip addr
```

확인한 내용은 다음과 같다.

- `whoami`: 현재 로그인한 사용자가 누구인지 확인
- `hostname`: 접속한 서버의 이름 확인
- `pwd`: 현재 작업 중인 디렉터리 확인
- `ip addr`: 네트워크 인터페이스와 VM의 IP 주소 확인

서버에서 작업할 때는 먼저 **누구로 로그인했는지, 어느 서버인지, 어느 위치인지** 확인하는 것이 중요하다.

## 파일과 디렉터리 실습

오늘 사용한 명령어는 다음과 같다.

| 명령어 | 실습에서 사용한 목적 |
| --- | --- |
| `pwd` | 현재 위치 확인 |
| `ls` | 현재 위치의 파일과 디렉터리 확인 |
| `cd` | 다른 디렉터리로 이동 |
| `mkdir` | 실습 및 백업 디렉터리 생성 |
| `touch` | 빈 실습 파일 생성 |
| `cp` | 원본을 유지한 채 파일 복사 |
| `mv` | 파일 이동 또는 이름 변경 |
| `rm` | 불필요한 파일 삭제 |

경로를 사용할 때 다음 차이도 확인했다.

- `/home/linuxuser/test.txt`: `/`부터 시작하는 **절대경로**
- `backup/test1.txt`: 현재 위치를 기준으로 찾는 **상대경로**
- `~`: 현재 사용자의 홈 디렉터리
- `..`: 현재 위치의 상위 디렉터리

## 직접 실습 - 파일과 디렉터리

홈 디렉터리 아래에 `linux-test`를 만들고 파일을 조작했다.

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

최종 구조는 다음과 같다.

```text
~/linux-test/
├── backup/
│   └── test1.txt
├── config.txt
└── test1.txt
```

`cp test1.txt backup/`은 원본을 남겨두고 `backup/`에 복사본을 만들었다. `mv test2.txt config.txt`는 같은 디렉터리 안에서 파일 이름을 변경했다.

## 발생한 문제

처음에는 홈 디렉터리에서 `linux-test`를 만든 뒤 그 안으로 이동하지 않고 파일을 생성했다.

```bash
mkdir linux-test
touch test1.txt test2.txt
```

명령어 자체는 정상 실행됐지만, `test1.txt`와 `test2.txt`가 의도한 `linux-test` 안이 아니라 홈 디렉터리에 만들어졌다.

## 원인과 해결

원인은 파일을 만들기 전에 현재 위치를 확인하지 않은 것이었다. `pwd`와 `ls`로 위치와 파일을 확인한 뒤 잘못 만든 파일을 삭제하고 올바른 위치에서 다시 생성했다.

```bash
pwd
ls
rm test1.txt test2.txt
cd linux-test
touch test1.txt test2.txt
ls
```

이 경험을 통해 명령어가 정확해도 현재 디렉터리가 다르면 전혀 다른 결과가 생긴다는 것을 확인했다.

## 파일 내용 확인과 출력

파일을 읽거나 로그를 확인하기 위해 다음 명령을 실습했다.

| 명령어 | 용도 |
| --- | --- |
| `cat` | 짧은 파일 전체 내용 출력 |
| `less` | 긴 파일을 페이지 단위로 확인하고 검색 |
| `head` | 파일 앞부분 확인 |
| `tail` | 파일 마지막 부분 확인 |
| `tail -f` | 파일에 새로 추가되는 내용을 실시간 확인 |
| `echo` | 문자열 출력 또는 파일에 내용 기록 |
| `history` | 이전에 실행한 명령어 확인 |

실습용 로그 파일을 직접 만들었다.

```bash
cd ~/linux-test
echo "INFO server started" > server.log
echo "INFO user connected" >> server.log
echo "WARNING disk usage 80%" >> server.log
echo "ERROR database connection failed" >> server.log
echo "INFO retry connection" >> server.log
```

내용을 여러 방식으로 확인했다.

```bash
cat server.log
head -n 2 server.log
tail -n 2 server.log
less server.log
history
```

`less`에서는 `/ERROR`로 원하는 문자열을 검색하고 `q`로 종료하는 방법도 확인했다.

## `>`와 `>>` 차이

리다이렉션 기호의 동작 차이를 실습했다.

```bash
echo "hello" > test.txt
echo "world" >> test.txt
```

- `>`: 기존 파일 내용을 **덮어쓰고** 새 내용을 기록
- `>>`: 기존 내용은 유지하고 파일 마지막에 **추가**

운영 환경에서는 `>`를 잘못 사용하면 설정 파일이나 로그 내용을 잃을 수 있으므로 특히 주의해야 한다.

## 장애 상황으로 이해한 내용

예를 들어 원래 설정 파일에 다음 내용이 있다고 가정한다.

```text
worker_processes auto;
events {}
```

이 상태에서 아래 명령을 실행하면,

```bash
echo "test" > nginx.conf
```

기존 설정은 사라지고 `test`만 남게 된다.

원인은 `>`가 append가 아니라 overwrite 동작을 하기 때문이다.

작업 전에는 파일 내용을 먼저 확인하고, 중요한 설정 파일이라면 백업본을 만든 뒤 수정하는 습관이 필요하다.

```bash
cat nginx.conf
cp nginx.conf nginx.conf.bak
```

실제 서비스 설정 변경에서는 앞으로 다음 절차를 기본으로 사용한다.

```text
내용 확인 → 백업 → 수정 → 설정 검증 → 서비스 반영 → 로그 확인
```

## 배운 점

- Linux 작업은 항상 현재 위치를 기준으로 실행된다.
- 파일 조작 전 `pwd`와 `ls`로 위치와 대상을 먼저 확인해야 한다.
- `cp`는 원본을 남기고, `mv`는 원본의 위치나 이름을 바꾼다.
- 절대경로는 위치와 관계없이 같은 대상을 가리키고, 상대경로는 현재 위치에 따라 대상이 달라진다.
- 짧은 파일은 `cat`, 긴 파일은 `less`, 최신 로그는 `tail`과 `tail -f`가 유용하다.
- `>`는 덮어쓰기, `>>`는 추가이므로 설정 파일 작업 전에 반드시 구분해야 한다.
- `rm`, `mv`, `cp`, 리다이렉션처럼 파일에 영향을 주는 작업은 실행 전후 확인이 중요하다.

앞으로 다음 순서를 기본 작업 습관으로 사용한다.

```text
현재 위치 확인 → 대상/내용 확인 → 필요 시 백업 → 명령 실행 → 결과 재확인
```
