# Day 0 — VirtualBox와 Ubuntu Server 실습 환경 구축

> Windows PC 위에 Ubuntu Server VM을 만들고 NAT 포트 포워딩과 SSH를 구성했다. 단순 설치 절차뿐 아니라 **Host/Guest/Hypervisor, OS와 Kernel, IP/Port, NAT, SSH Client/Server, Listen**이 서로 어떻게 연결되는지 이해하는 것을 목표로 한다.

## 📌 이번에 배운 내용

- Operating System, Kernel, Linux Distribution
- Host OS / Guest OS / VM / Hypervisor
- VirtualBox가 하는 일
- CPU / Memory / Disk / Network를 VM에 할당하는 이유
- Ubuntu Server와 Ubuntu Desktop
- IP Address / Port / Socket의 기초
- NAT와 Port Forwarding
- `127.0.0.1` Loopback
- SSH Client / SSH Server / `sshd`
- `ssh -p`, `ip addr`, `systemctl status ssh`
- SSH 접속 장애를 단계별로 확인하는 방법

## 📚 목차

1. OS, Kernel, Linux, Ubuntu
2. 가상화와 VM
3. VM 자원
4. 네트워크의 IP와 Port
5. NAT와 Port Forwarding
6. SSH
7. 주요 명령어와 옵션
8. 실제 실습
9. 헷갈리기 쉬운 부분
10. Troubleshooting
11. 실무 포인트
12. 핵심 정리
13. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 옵션/인자 | 의미 | 실무 사용 |
|---|---|---|---|
| `whoami` | 없음 | 현재 명령을 실행하는 사용자 이름 확인 | 서버/계정 확인 |
| `hostname` | 없음 | 현재 호스트 이름 확인 | 여러 서버 작업 시 대상 서버 확인 |
| `pwd` | 없음 | 현재 작업 디렉터리 확인 | 잘못된 경로에서 작업 방지 |
| `ip addr` | `addr` = address 객체 | 인터페이스와 IP 주소 확인 | 네트워크 점검 |
| `systemctl status ssh` | `status` = Unit 상태 조회 | SSH 서비스 상태 확인 | 접속 장애 분석 |
| `systemctl start ssh` | `start` = 지금 시작 | SSH 서비스 시작 | 정지된 서비스 기동 |
| `systemctl enable ssh` | `enable` = 부팅 자동 시작 설정 | 다음 부팅에도 자동 시작 | 서버 운영 설정 |
| `ssh -p 2222 linuxuser@127.0.0.1` | `-p` = port | TCP 2222번으로 SSH 접속 | 비기본 포트 접속 |

---

## 1. OS, Kernel, Linux, Ubuntu

### 운영체제(OS)란?

운영체제는 하드웨어 자원을 관리하고 프로그램이 실행될 수 있는 환경을 제공하는 시스템 소프트웨어다. CPU, Memory, Disk, Device, Process, File System 등을 직접 또는 간접적으로 관리한다.

### Kernel이란?

Kernel은 운영체제의 핵심 부분으로, 사용자 프로그램과 하드웨어 사이에서 자원을 관리한다.

```text
Application / Shell / Utilities
          ↓
       Kernel
          ↓
CPU / Memory / Disk / Network Device
```

### Linux와 Ubuntu의 관계

Linux는 엄밀히 말하면 **Linux Kernel**을 뜻한다. Ubuntu는 Linux Kernel에 패키지 관리자, 사용자 공간 도구, 설치 프로그램 등을 결합한 **Linux Distribution(배포판)**이다.

```text
Linux Kernel
├─ Ubuntu
├─ Debian
├─ Rocky Linux
└─ Fedora
```

즉 `Windows도 OS고 Linux도 OS냐?`라는 질문에는 실무적으로는 Linux를 OS 계열로 부르지만, 더 정확히는 **Linux Kernel을 중심으로 한 배포판들이 실제 사용되는 운영체제**라고 이해하면 된다.

---

## 2. 가상화와 VM

### Host와 Guest

- **Host**: 실제 물리 컴퓨터와 그 위에서 실행되는 기본 OS. 현재 Windows PC.
- **Guest**: VM 안에서 실행되는 OS. 현재 Ubuntu Server.
- **VM(Virtual Machine)**: 가상 CPU, RAM, Disk, NIC를 가진 소프트웨어 컴퓨터.
- **Hypervisor**: 물리 자원을 VM에 제공하고 여러 VM을 격리·실행하는 계층.

현재 구조:

```text
Physical PC
└─ Windows (Host OS)
   └─ VirtualBox (Hypervisor)
      └─ Ubuntu Server VM (Guest OS)
         └─ Linux Kernel
```

### VM을 쓰는 이유

서버 운영 학습에서는 OS 설치, 네트워크 변경, 사용자/권한 실습, 서비스 중지 같은 위험한 작업이 많다. VM은 Host와 격리된 환경을 제공하므로 반복 실습과 복구가 쉽다.

### VM과 Container 차이

VM은 Guest OS와 별도 Kernel을 실행한다. 일반적인 Container는 Host Kernel을 공유하고 Process를 격리한다. 현재는 Linux 운영체제 자체를 학습하므로 VM이 적합하다.

---

## 3. VM 자원

VM에 CPU, RAM, Disk를 할당한다는 것은 물리 자원의 일부를 Guest가 사용할 수 있게 하는 것이다.

| 자원 | 의미 | 부족하면 |
|---|---|---|
| vCPU | Guest가 사용할 CPU 실행 자원 | 명령/서비스 처리 지연 |
| RAM | 실행 중 프로세스와 Kernel이 사용하는 메모리 | Swap 증가, OOM 위험 |
| Virtual Disk | OS·패키지·로그·파일 저장 | Disk Full 장애 |
| Virtual NIC | VM의 네트워크 인터페이스 | 외부/Host 통신 불가 |

동적 할당 디스크는 최대 크기를 미리 정하지만 Host 파일은 실제 사용량에 따라 커질 수 있다.

---

## 4. IP Address와 Port

### IP Address

네트워크에서 Host/Interface를 식별하기 위한 주소다. 한 서버에 여러 NIC와 여러 IP가 존재할 수도 있다.

### Port

하나의 IP 안에서 어떤 Application/Service로 연결할지 구분하는 번호다. TCP/UDP Port는 0~65535 범위다.

```text
IP Address → 어느 Host인가?
Port       → 그 Host의 어느 Service인가?
```

예:

```text
192.168.0.10:22
```

`192.168.0.10`은 서버 주소, `22`는 SSH Server가 사용하는 대표 Port다.

### Listen이란?

서버 프로그램이 특정 IP/Port에서 연결 요청을 기다리는 상태를 흔히 **listen 중**이라고 한다. 서비스 프로세스가 실행 중이어도 해당 Port에서 Listen하지 않는다면 Client 접속은 실패할 수 있다.

---

## 5. NAT와 Port Forwarding

### NAT

NAT(Network Address Translation)는 한 네트워크의 주소를 다른 주소 체계로 변환하는 기술이다. VirtualBox NAT에서는 Guest가 Host를 통해 외부로 나가는 통신이 가능하다.

### 왜 Port Forwarding이 필요한가?

NAT 뒤의 Guest는 Host 외부에서 바로 접근하기 어렵다. 그래서 Host의 특정 Port로 들어온 연결을 Guest의 특정 Port로 전달한다.

현재 구성:

```text
Windows SSH Client
        ↓
127.0.0.1:2222
        ↓
VirtualBox NAT Port Forwarding
        ↓
Ubuntu Guest:22
        ↓
OpenSSH Server
```

`Host Port 2222`와 `Guest Port 22`는 같을 필요가 없다.

### `127.0.0.1`

IPv4 Loopback 주소다. 현재 Host 자기 자신을 가리킨다. Host IP를 `127.0.0.1`로 두면 일반적으로 같은 Windows PC에서만 해당 포워딩에 접근하도록 제한하는 효과가 있다.

---

## 6. SSH

### SSH란?

SSH(Secure Shell)는 암호화된 네트워크 연결을 통해 원격 시스템에서 Shell을 사용할 수 있게 하는 Protocol이다.

```text
SSH Client
→ TCP Connection
→ SSH Server(sshd)
→ Authentication
→ Remote Shell
```

현재 Windows의 `ssh` 명령은 Client이고 Ubuntu의 OpenSSH Server가 Server다.

### SSH와 Shell은 같은 것인가?

아니다. SSH는 **원격 연결 Protocol**이고 Bash는 접속 후 명령을 해석하는 **Shell**이다.

---

## 7. 주요 명령어와 옵션

### `ip addr`

`ip`는 Linux 네트워크 객체를 조회·관리하는 명령이다. `addr`는 address 객체를 뜻한다.

```bash
ip addr
```

주요하게 볼 것:

```text
인터페이스 이름
state UP/DOWN
inet IPv4주소/Prefix
```

### `ssh -p`

```bash
ssh -p 2222 linuxuser@127.0.0.1
```

구조:

```text
ssh             → SSH Client 실행
-p 2222         → 접속할 TCP Port 지정
linuxuser       → 원격 로그인 사용자
127.0.0.1       → 접속할 Host 주소
```

`-p`는 **port** 옵션이다. 대문자 `-P`와 혼동하지 않는다. OpenSSH `ssh`에서 접속 Port는 소문자 `-p`다.

### `systemctl status/start/enable`

- `status`: 현재 Unit 상태 확인
- `start`: 현재 세션에서 서비스 시작
- `enable`: 부팅 시 자동 시작 연결 생성

```text
start ≠ enable
```

실행 중인지와 부팅 자동 시작 여부는 별개의 상태다.

---

## 8. 🧪 실제 실습

```bash
whoami
hostname
pwd
ip addr
sudo systemctl status ssh
```

VirtualBox NAT Port Forwarding:

```text
Host IP   : 127.0.0.1
Host Port : 2222
Guest Port: 22
Protocol  : TCP
```

Windows PowerShell:

```bash
ssh -p 2222 linuxuser@127.0.0.1
```

접속 후 프롬프트가 Ubuntu 계정으로 바뀌는 것을 확인했다.

---

## 9. ⚠️ 헷갈리기 쉬운 부분

> ⚠️ VM을 만들었다고 Guest OS가 자동으로 설치된 것은 아니다. VM은 가상 하드웨어이고 ISO는 OS 설치 미디어다.

> ⚠️ `127.0.0.1:2222`는 Ubuntu 자체 주소가 아니라 Host 측 접점이다. VirtualBox가 Guest의 22번 Port로 전달한다.

> ⚠️ Ubuntu를 SSH로 접속한다고 Ubuntu가 Windows 안의 PowerShell 프로그램으로 바뀌는 것이 아니다. Ubuntu는 VM 안에서 계속 실행되고 PowerShell은 원격 Client 역할만 한다.

---

## 10. 🔧 Troubleshooting

### 증상: SSH 접속 실패

확인 흐름:

```text
VM이 실행 중인가?
→ Guest OS가 부팅됐는가?
→ Guest NIC/IP가 정상인가?
→ ssh 서비스가 active인가?
→ Guest 22번에서 Listen하는가?
→ NAT Port Forwarding이 정확한가?
→ Client가 127.0.0.1:2222로 접속하는가?
→ 사용자 이름/인증이 맞는가?
```

현재 단계 명령:

```bash
ip addr
sudo systemctl status ssh
```

Client:

```bash
ssh -p 2222 linuxuser@127.0.0.1
```

운영에서는 추측으로 설정을 바꾸기보다 **연결 경로를 한 단계씩 검증**한다.

---

## 11. 💼 실무 포인트

VirtualBox 실습은 Cloud VM과 구조적으로 연결된다. 실제 업무에서도 VM의 CPU/RAM/Disk/NIC, IP, Port, Firewall/Security Group, SSH Service, 인증을 단계적으로 확인한다.

장애 확인 사고방식:

```text
Client
→ Address/Port
→ Network Path
→ Firewall/NAT
→ Server Listen
→ Process/Service
→ Authentication
```

이 순서를 익히면 SSH뿐 아니라 Web/DB 접속 장애 분석에도 그대로 확장할 수 있다.

---

## 12. ✅ 핵심 정리

- Ubuntu는 Linux Kernel을 사용하는 Linux 배포판이다.
- Host는 Windows, Guest는 Ubuntu Server다.
- VirtualBox는 VM을 실행하는 Hypervisor 역할을 한다.
- IP는 Host/Interface를, Port는 Service Endpoint를 구분한다.
- NAT 환경에서는 Host→Guest 접근을 위해 Port Forwarding을 사용할 수 있다.
- 현재 구성은 `127.0.0.1:2222 → Guest:22`다.
- SSH는 원격 접속 Protocol이고 Bash는 접속 후 사용하는 Shell이다.
- 서비스 실행 상태와 부팅 자동 시작 상태는 서로 다르다.

---

## 13. 🧠 복습 문제

1. Kernel과 Linux Distribution의 차이는 무엇인가?
2. Host OS, Guest OS, Hypervisor를 현재 환경 기준으로 설명해보라.
3. IP와 Port는 각각 무엇을 식별하는가?
4. `127.0.0.1`은 무엇인가?
5. `2222 → 22` Port Forwarding의 흐름을 설명해보라.
6. SSH Client와 SSH Server는 각각 어디에서 동작하는가?
7. `systemctl start ssh`와 `systemctl enable ssh`의 차이는 무엇인가?
8. SSH 접속 실패 시 어떤 순서로 점검할 것인가?
