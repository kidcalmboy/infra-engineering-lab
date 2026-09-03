# Day 0 - VirtualBox와 Ubuntu Server 실습 환경 구축

> Windows PC 안에 Ubuntu Server 가상 머신을 구성하고, NAT 포트 포워딩과 SSH를 이용해 실제 원격 서버처럼 접속하는 실습 환경을 만들었다.

## 📌 이번에 배운 내용

- Host OS / Guest OS / VM / Hypervisor
- Ubuntu Server와 Ubuntu Desktop의 차이
- CPU, Memory, Disk를 VM에 할당하는 이유
- NAT 네트워크와 포트 포워딩
- `127.0.0.1`, SSH 22번 포트, Host Port와 Guest Port
- OpenSSH Server 상태 확인
- Windows PowerShell에서 Ubuntu Server로 SSH 접속
- 부팅 가능한 OS가 없을 때 ISO 연결 여부 확인

## 📚 목차

1. 가상화와 VM
2. Ubuntu Server를 선택한 이유
3. VM 자원 구성
4. SSH 개념
5. NAT와 포트 포워딩
6. 실습 환경 구축
7. 헷갈리기 쉬운 부분
8. Troubleshooting
9. 실무 포인트
10. 핵심 정리
11. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 용도 |
|---|---|
| `whoami` | 현재 로그인 사용자 확인 |
| `hostname` | 현재 서버 이름 확인 |
| `pwd` | 현재 작업 디렉터리 확인 |
| `ip addr` | 네트워크 인터페이스와 IP 확인 |
| `sudo systemctl status ssh` | SSH 서비스 상태 확인 |
| `sudo systemctl start ssh` | SSH 서비스 시작 |
| `sudo systemctl enable ssh` | 부팅 시 SSH 자동 시작 설정 |
| `ssh -p 2222 linuxuser@127.0.0.1` | Windows에서 VM의 SSH 서버로 접속 |

---

## 1. 가상화와 VM

### 한 줄 정의

가상화는 **한 대의 물리 컴퓨터 자원을 나누어 별도의 소프트웨어 컴퓨터를 실행하는 기술**이다.

실습 구조:

```text
Windows PC
└─ VirtualBox
   └─ Ubuntu Server VM
      └─ OpenSSH Server
```

### 주요 용어

| 용어 | 의미 |
|---|---|
| Host OS | 실제 컴퓨터에서 실행되는 운영체제. 이번 환경에서는 Windows |
| Guest OS | VM 안에 설치한 운영체제. 이번 환경에서는 Ubuntu Server |
| VM | Virtual Machine. 가상 CPU·메모리·디스크·네트워크를 가진 소프트웨어 컴퓨터 |
| Hypervisor | VM을 실행하고 물리 자원을 가상 머신에 할당하는 계층 |

### VM을 사용하는 이유

- Windows를 유지하면서 Linux 서버를 실습할 수 있다.
- 설정을 잘못해도 Host OS에 직접 미치는 위험을 줄일 수 있다.
- 서버 구축·복구를 반복 연습하기 쉽다.
- CPU, 메모리, 디스크, 네트워크를 별도 서버처럼 구성할 수 있다.
- 이후 클라우드 VM 환경을 이해하는 기초가 된다.

### VM과 컨테이너 차이

VM은 Guest OS와 자체 커널을 실행하는 반면, 일반적인 컨테이너는 Host의 커널을 공유한다. 현재 단계에서는 **Ubuntu Server 운영체제 자체를 학습**해야 하므로 VM이 적합하다.

---

## 2. Ubuntu Server를 선택한 이유

### 한 줄 정의

Ubuntu Server는 GUI 의존도를 낮추고 CLI 중심으로 서버 운영을 학습하기 좋은 Linux 배포판이다.

Ubuntu는 Linux 커널을 기반으로 만들어진 여러 Linux 배포판 중 하나다.

```text
Linux Kernel
   ↓
Ubuntu / Debian / Rocky Linux / etc.
```

### Server와 Desktop

Ubuntu Desktop은 GUI 환경을 기본 제공하지만 Ubuntu Server는 서버 운영에 필요한 CLI 중심 환경으로 구성된다.

이번 학습 목표는 실제 서버처럼 명령어, 서비스, 로그, 네트워크를 직접 다루는 것이므로 Ubuntu Server를 사용했다.

### LTS

LTS는 **Long Term Support**의 약자다. 장기간 보안 업데이트와 유지보수를 제공하므로 운영 환경 학습에 적합하다.

---

## 3. VM 자원 구성

실습 VM 예시:

| 항목 | 설정 |
|---|---|
| CPU | 2 Core |
| Memory | 4 GB |
| Disk | 30 GB, 동적 할당 |
| Network | NAT |

### CPU

VM이 사용할 가상 CPU 자원이다. 너무 적으면 VM이 느리고, 너무 많이 주면 Host Windows가 느려질 수 있다.

### Memory

Ubuntu Server와 이후 실행할 서비스가 사용하는 메모리다.

### Disk

Guest OS와 패키지, 로그, 실습 파일이 저장되는 가상 디스크다.

### 동적 할당

30GB로 설정했다고 해서 Host 디스크에서 처음부터 30GB를 전부 차지하는 것은 아니다. 실제 VM 사용량에 따라 가상 디스크 파일이 증가한다.

---

## 4. SSH란 무엇인가

### 한 줄 정의

SSH는 **네트워크를 통해 다른 컴퓨터의 셸에 안전하게 원격 접속하기 위한 프로토콜**이다.

이번 환경에서는:

```text
Windows
→ SSH Client
→ Ubuntu Server의 OpenSSH Server
```

구조로 사용한다.

### Client와 Server

| 구분 | 역할 |
|---|---|
| SSH Client | 접속을 요청하는 프로그램. Windows의 `ssh` 명령 |
| SSH Server | 접속 요청을 받아 셸을 제공. Ubuntu의 OpenSSH Server |

SSH 서버의 기본 TCP 포트는 `22`다.

상태 확인:

```bash
sudo systemctl status ssh
```

정상 상태 예:

```text
Active: active (running)
```

---

## 5. NAT와 포트 포워딩

### NAT 한 줄 정의

NAT는 VM이 Host의 네트워크를 통해 외부와 통신할 수 있도록 주소를 변환하는 방식이다.

VirtualBox NAT를 사용하면 VM에서 인터넷으로 나가는 연결은 비교적 간단하지만, Host에서 Guest 서비스로 들어가는 연결은 별도 설정이 필요할 수 있다.

그래서 **포트 포워딩**을 설정했다.

### 포트 포워딩

```text
Windows 127.0.0.1:2222
        ↓
VirtualBox Port Forwarding
        ↓
Ubuntu Guest:22
        ↓
OpenSSH Server
```

설정 예:

| 항목 | 값 |
|---|---|
| Protocol | TCP |
| Host IP | `127.0.0.1` |
| Host Port | `2222` |
| Guest Port | `22` |

### `127.0.0.1`

현재 컴퓨터 자신을 가리키는 **Loopback 주소**다.

Host IP를 `127.0.0.1`로 제한하면 같은 Windows PC에서 해당 포트 포워딩을 사용할 수 있다.

### 왜 `2222 → 22`인가?

Host 포트와 Guest 서비스 포트는 같을 필요가 없다.

```text
Host 2222
→ Guest 22
```

처럼 연결할 수 있다. 여러 VM을 운영하면 Host Port를 `2222`, `2223` 등으로 나누어 각 VM의 SSH 22번 포트로 연결할 수도 있다.

---

## 6. 🧪 실습 환경 구축

### 1. Ubuntu Server 설치

VirtualBox VM에 Ubuntu Server ISO를 연결하고 설치했다.

주요 구성:

```text
Network → DHCP
Storage → 자동 구성
OpenSSH Server → 설치
일반 사용자 계정 생성
```

### 2. 설치 결과 확인

```bash
whoami
hostname
pwd
ip addr
```

### 3. SSH 서비스 확인

```bash
sudo systemctl status ssh
```

필요한 경우:

```bash
sudo systemctl start ssh
sudo systemctl enable ssh
```

### 4. VirtualBox NAT 포트 포워딩

```text
Host IP   : 127.0.0.1
Host Port : 2222
Guest Port: 22
```

### 5. Windows PowerShell에서 접속

```bash
ssh -p 2222 linuxuser@127.0.0.1
```

`-p`는 SSH Client가 접속할 **port**를 지정한다.

접속 후 프롬프트가 Ubuntu 서버 계정으로 변경되는 것을 확인했다.

---

## 7. ⚠️ 헷갈리기 쉬운 부분

### Ubuntu와 Linux는 같은 말인가?

정확히는 아니다.

Linux는 커널을 중심으로 한 운영체제 생태계를 의미하고, Ubuntu는 Linux 커널을 사용하는 **Linux 배포판** 중 하나다.

### SSH를 Windows에 설치해서 Ubuntu를 Windows로 바꾼 것인가?

아니다.

Ubuntu Server는 VirtualBox VM 안에서 계속 실행되고 있다. Windows의 PowerShell은 SSH Client로 그 Ubuntu Server의 셸에 원격 접속하는 것이다.

### `127.0.0.1:2222`가 Ubuntu의 실제 IP와 포트인가?

아니다.

`127.0.0.1:2222`는 Windows Host 쪽 접점이고, VirtualBox가 이를 Guest의 SSH 22번 포트로 전달한다.

### GUI를 설치해야 복사·붙여넣기를 할 수 있는가?

반드시 그렇지 않다. 서버 운영 학습에서는 GUI를 추가하기보다 SSH로 Windows Terminal/PowerShell에서 접속하면 편한 터미널 기능을 사용할 수 있다.

---

## 8. 🔧 Troubleshooting

### 문제 1 - VM 부팅 시 운영체제를 찾을 수 없음

### 증상

VM을 실행했지만 부팅 가능한 운영체제가 없다는 메시지가 나타났다.

### 원인

Ubuntu Server ISO가 가상 광학 드라이브에 연결되지 않았다.

### 해결

VirtualBox 저장소 설정에서 Ubuntu Server ISO를 연결한 뒤 다시 부팅했다.

### 배운 점

```text
VM 생성
≠
Guest OS 설치 완료
```

VM이라는 가상 하드웨어를 만든 뒤 설치 미디어를 연결해 운영체제를 설치해야 한다.

### 문제 2 - SSH 접속이 되지 않음

확인 순서:

```text
VM이 실행 중인가?
→ Ubuntu에 IP가 있는가?
→ SSH 서비스가 running 상태인가?
→ 포트 포워딩이 맞는가?
→ Host Port가 2222인가?
→ 사용자 이름이 맞는가?
```

명령:

```bash
ip addr
sudo systemctl status ssh
```

Windows:

```bash
ssh -p 2222 linuxuser@127.0.0.1
```

### 배운 점

네트워크 장애는 한 번에 추측하기보다 **연결 경로를 단계별로 확인**한다.

---

## 9. 💼 실무 포인트

SSH 장애를 분석할 때 다음 흐름을 습관화한다.

```text
Client
→ 접속 주소/포트
→ Network/NAT/Firewall
→ Server Listen Port
→ SSH Service
→ User Authentication
```

현재 실습 환경은 VirtualBox지만 이 사고방식은 클라우드 VM, 사내 서버, 원격 Linux 서버에서도 그대로 활용된다.

또한 서버는 GUI보다 SSH 기반 원격 운영이 일반적이므로 CLI에 익숙해지는 것이 중요하다.

---

## 10. ✅ 핵심 정리

- Host OS는 Windows, Guest OS는 Ubuntu Server다.
- VirtualBox는 VM을 만들고 실행하는 가상화 도구다.
- Ubuntu는 Linux 배포판 중 하나다.
- SSH는 원격으로 서버 셸에 접속하는 프로토콜이며 기본 포트는 22다.
- NAT에서는 Host에서 Guest로 들어오는 연결을 위해 포트 포워딩을 사용할 수 있다.
- `127.0.0.1:2222 → Guest:22` 구조로 SSH 접속을 구성했다.
- SSH 문제는 VM → IP → 서비스 → 포트 포워딩 → 계정 순서로 확인한다.

---

## 11. 🧠 복습 문제

1. Host OS와 Guest OS의 차이는 무엇인가?
2. Ubuntu와 Linux의 관계를 설명해보라.
3. VM과 컨테이너의 가장 큰 구조적 차이는 무엇인가?
4. SSH Client와 SSH Server는 각각 어디에서 동작하는가?
5. SSH의 기본 포트 번호는 무엇인가?
6. `ssh -p 2222 linuxuser@127.0.0.1`에서 `2222`는 어느 쪽 포트인가?
7. NAT 환경에서 포트 포워딩이 필요한 이유는 무엇인가?
8. SSH 접속 실패 시 어떤 순서로 확인해야 하는가?
