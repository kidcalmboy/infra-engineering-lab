## 학습 도구 및 기타 팁
### Markdown
markdown .md 파일은 줄 바꿈을 하려면 문장 끝에 공백 2개(enter나 space)를 넣거나 br 사용하기.

### Markdown 미리보기
VS Code에서 `Ctrl + Shift + V`를 누르면 Markdown 미리보기가 열려서 GitHub에 어떻게 표시될지 업로드 전에 확인 가능.

### Bash Here Document로 여러 줄 파일 만들기
여러 줄의 내용을 한 번에 파일에 저장할 때 `cat`과 Here Document(`<<`)를 함께 사용할 수 있다.

```bash
cat > access.log <<'EOF'
10.0.0.1 GET /index.html 200 120
10.0.0.2 POST /login 401 80
10.0.0.3 GET /login 500 150
EOF
```

동작 흐름:

```text
cat
→ 표준 입력을 받음

<<'EOF'
→ EOF가 나올 때까지 여러 줄을 입력으로 전달

> access.log
→ 전달받은 내용을 access.log에 저장
```

`EOF`는 입력 종료를 표시하는 구분자이며 반드시 `EOF`라는 이름일 필요는 없다.

```bash
cat > test.txt <<'END'
hello
linux
server
END
```

`>`는 기존 파일을 덮어쓰고, `>>`는 기존 내용 뒤에 추가한다.

```bash
cat >> test.txt <<'EOF'
new line 1
new line 2
EOF
```

정리:

```text
cat > file <<'EOF'   → 여러 줄 입력 후 파일 덮어쓰기
cat >> file <<'EOF'  → 여러 줄 입력 후 기존 파일 뒤에 추가
```

실습용 로그 파일이나 설정 샘플을 여러 줄로 빠르게 만들 때 유용하다.

---

### Linux, Ubuntu, VM 관계

#### Linux와 Ubuntu
Linux는 엄밀히 말하면 운영체제의 핵심인 **Linux Kernel**을 의미한다. Ubuntu는 Linux Kernel에 GNU 도구, 패키지 관리자, systemd, 기본 프로그램 등을 묶어 실제로 사용할 수 있도록 만든 **Linux 배포판(Distribution)** 중 하나다.

```text
Linux Kernel
├─ Ubuntu
├─ Debian
├─ Rocky Linux
├─ AlmaLinux
├─ Fedora
└─ Arch Linux
```

Ubuntu는 Debian 계열의 Linux 배포판이다.

```text
Debian
  ↓
Ubuntu
```

배포판마다 패키지 관리자 등 일부 요소는 다르지만, `ls`, `cd`, `cp`, `mv`, `grep`, `chmod`, `ps`, `systemctl` 같은 Linux 기본 개념과 명령어는 대부분 공통으로 사용할 수 있다.

예시:

```text
Ubuntu / Debian   → apt
RHEL / Rocky      → dnf
Arch Linux        → pacman
```

#### Host OS와 Guest OS
VirtualBox 같은 가상화 프로그램을 사용하면 실제 PC 안에 가상의 컴퓨터를 만들고 다른 운영체제를 실행할 수 있다.

현재 실습 환경:

```text
Windows 노트북
└─ VirtualBox
   └─ Ubuntu Server
      └─ Linux Kernel
```

- **Host OS**: 실제 컴퓨터에서 실행 중인 운영체제
- **Guest OS**: 가상 머신(VM) 안에서 실행되는 운영체제

현재 기준:

```text
Host OS  = Windows
Guest OS = Ubuntu Server
```

VM에는 Ubuntu뿐만 아니라 Windows Server, Rocky Linux 등 다른 운영체제도 설치할 수 있다.

```text
Windows Host
└─ VirtualBox
   ├─ Ubuntu Server VM
   ├─ Rocky Linux VM
   └─ Windows Server VM
```

VirtualBox는 CPU, RAM, 디스크, 네트워크 등의 물리 자원을 가상 머신에 할당하여 Guest OS가 독립된 컴퓨터처럼 동작하게 한다.

---

### SSH 개념

#### SSH란?
SSH는 **Secure Shell**의 약자로, 네트워크를 통해 다른 컴퓨터의 Shell(터미널)에 안전하게 원격 접속하기 위한 프로토콜이다.

쉽게 표현하면:

```text
내 컴퓨터의 터미널
       │
       │ SSH
       ▼
원격 Linux 서버의 터미널
```

실제 서버 관리에서는 서버 앞에 직접 가서 키보드로 명령어를 입력하기보다 SSH를 통해 원격으로 접속해서 관리하는 경우가 일반적이다.

#### SSH Client와 SSH Server
SSH는 접속하는 쪽과 접속을 받는 쪽으로 나뉜다.

```text
Windows
SSH Client
   │
   │ SSH 연결
   ▼
Ubuntu Server
SSH Server (sshd)
```

- **SSH Client**: 원격 서버에 접속하는 프로그램
- **SSH Server**: 외부 SSH 접속을 받아주는 프로그램
- **sshd**: Ubuntu에서 동작하는 SSH 서버 데몬

Ubuntu에 SSH 서버 설치:

```bash
sudo apt install openssh-server -y
```

SSH 서비스 실행 및 부팅 시 자동 시작:

```bash
sudo systemctl enable --now ssh
```

상태 확인:

```bash
systemctl status ssh
```

#### SSH 기본 포트
SSH는 기본적으로 **TCP 22번 포트**를 사용한다.

일반적인 접속:

```bash
ssh username@192.168.0.10
```

위 명령은 사실상 다음과 같다.

```bash
ssh -p 22 username@192.168.0.10
```

`22`가 SSH의 기본 포트이기 때문에 생략할 수 있다.

---

### VirtualBox NAT와 SSH 포트 포워딩
VirtualBox의 Ubuntu VM을 NAT 방식으로 사용할 때 Windows에서 VM의 SSH 22번 포트로 접근하기 위해 포트 포워딩을 설정할 수 있다.

실습 설정 예시:

```text
Windows                  Ubuntu VM
127.0.0.1:2222  ──────→  TCP 22
```

VirtualBox 포트 포워딩 규칙:

```text
이름       : SSH
프로토콜   : TCP
호스트 IP  : 127.0.0.1
호스트 포트: 2222
게스트 포트: 22
```

Windows PowerShell 또는 Windows Terminal에서 접속:

```bash
ssh -p 2222 username@127.0.0.1
```

명령 구성:

```text
ssh
→ SSH Client 실행

-p 2222
→ Windows의 2222번 포트 사용

username
→ Ubuntu에 존재하는 사용자 계정

127.0.0.1
→ 현재 Windows 컴퓨터 자신(localhost)
```

VirtualBox가 Windows의 `127.0.0.1:2222`로 들어온 요청을 Ubuntu VM의 `22`번 포트로 전달한다.

---

### SSH 접속 후 명령어는 어디에서 실행되는가?
Windows PowerShell에서 SSH 접속 전에는 명령어가 Windows에서 실행된다.

```text
PowerShell
C:\Users\...
```

SSH 접속 후 다음처럼 프롬프트가 바뀌면:

```text
user@ubuntu-server:~$
```

이후 입력하는 Linux 명령어는 **Windows가 아니라 Ubuntu VM에서 실행된다.**

예를 들어 SSH 접속 후:

```bash
ls
```

를 실행하면 실제 흐름은 다음과 같다.

```text
Windows에서 "ls" 입력
        ↓
SSH Client
        ↓
네트워크
        ↓
Ubuntu의 sshd
        ↓
Ubuntu Shell (예: bash)
        ↓
ls 프로그램 실행
        ↓
Linux Kernel이 파일 시스템 처리
        ↓
결과를 SSH를 통해 Windows로 전송
        ↓
PowerShell 화면에 출력
```

따라서 PowerShell 화면을 사용하고 있어도 SSH 로그인 이후에는 Ubuntu Server를 원격 조작하고 있는 것이다.

---

### Shell과 Kernel의 관계
SSH가 Linux Kernel에 직접 연결되는 것은 아니다.

구조는 다음과 같이 이해하면 된다.

```text
Windows PowerShell
      │
      │ SSH
      ▼
Ubuntu SSH Server (sshd)
      │
      ▼
Shell (bash 등)
      │
      ▼
Linux 프로그램 / 명령어
      │
      ▼
Linux Kernel
      │
      ▼
CPU / RAM / Disk / Network
```

- **Shell**: 사용자가 명령어를 입력하고 결과를 확인하는 인터페이스
- **Kernel**: 프로세스, 메모리, 파일 시스템, 장치, 네트워크 등의 시스템 자원을 관리하는 운영체제의 핵심
- **SSH**: 원격지에서 Shell에 안전하게 접근할 수 있도록 해주는 통신 프로토콜

서버 운영 관점에서 이 구조를 이해하면 `ssh`, `bash`, `systemctl`, 프로세스, 포트, 네트워크의 관계를 이해하기 쉬워진다.
