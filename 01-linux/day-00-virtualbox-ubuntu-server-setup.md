# Day 0 - VirtualBox와 Ubuntu Server 실습 환경 구축

## 구축 목표

Windows PC 안에 Ubuntu Server 가상 머신을 만들고, Windows PowerShell에서 SSH로 접속할 수 있는 Linux 실습 환경을 구성한다.

```text
Windows PC
  └─ VirtualBox
      └─ Ubuntu Server VM
          └─ OpenSSH Server
```

## 사용 환경

- Host OS: Windows
- 가상화 도구: Oracle VirtualBox
- Guest OS: Ubuntu Server LTS
- 네트워크 방식: NAT
- 원격 접속: SSH

## 가상화 환경이란

가상화는 한 대의 물리 컴퓨터 안에 소프트웨어로 만든 별도의 컴퓨터를 실행하는 기술이다. 이번 실습에서는 실제 Windows PC의 CPU, 메모리, 디스크 일부를 VirtualBox가 가상 하드웨어처럼 Ubuntu Server에 제공한다.

```text
물리 컴퓨터
├── Windows: Host OS
└── VirtualBox: 가상 머신을 생성하고 실행하는 프로그램
    └── Ubuntu Server: Guest OS
```

- **Host OS**: 실제 컴퓨터에서 실행되는 운영체제이며 이번 환경에서는 Windows이다.
- **Guest OS**: 가상 머신 안에 설치한 운영체제이며 이번 환경에서는 Ubuntu Server이다.
- **VM**: Virtual Machine의 약자로, 가상 CPU·메모리·디스크·네트워크를 가진 소프트웨어 컴퓨터이다.
- **Hypervisor**: 물리 자원을 가상 머신에 나누어 주고 VM 실행을 관리하는 계층이다. VirtualBox는 Windows 위에서 동작하는 가상화 프로그램이다.

여기서 말하는 가상 환경은 Python 패키지를 분리하는 `venv`가 아니라, 운영체제 전체를 실행하는 **가상 머신 환경**이다. Docker 컨테이너도 격리된 환경을 제공하지만 Host의 커널을 공유한다는 점에서 별도의 운영체제를 실행하는 VM과 다르다.

## 가상 머신으로 실습하는 이유

Ubuntu를 실제 PC에 바로 설치하지 않고 VM을 사용한 이유는 다음과 같다.

- Windows를 유지한 상태에서 Linux 서버를 함께 실행할 수 있다.
- Linux 설정을 잘못하거나 파일을 삭제해도 Host OS에 미치는 영향을 줄일 수 있다.
- 문제가 생기면 VM을 다시 만들 수 있어 설치와 장애 복구를 반복해서 연습하기 쉽다.
- CPU, 메모리, 디스크, 네트워크를 독립된 서버처럼 설정할 수 있다.
- 이후 AWS EC2 같은 클라우드 가상 서버를 이해하는 기초가 된다.

완전히 실제 서버와 같지는 않지만, Linux 명령어, 사용자와 권한, 서비스, 네트워크, 로그, SSH를 학습하기에는 충분한 환경이다.

## VirtualBox와 Ubuntu Server 준비

VirtualBox는 Windows 안에 별도의 가상 컴퓨터를 만들기 위해 설치했다. Ubuntu는 GUI 중심의 Desktop 버전이 아닌 CLI 중심의 Server LTS ISO를 사용했다.

가상 머신은 다음 사양으로 구성했다.

| 항목 | 설정값 |
| --- | --- |
| CPU | 2 Core |
| Memory | 4 GB |
| Disk | 30 GB, 동적 할당 |
| Network | NAT |

PC 메모리가 부족하면 VM 메모리는 2 GB까지 낮출 수 있지만, 이후 Nginx와 간단한 데이터베이스 실습까지 고려해 4 GB를 선택했다.

### 설정값을 선택한 이유

- **CPU 2 Core**: 기본 Linux 명령과 서버 서비스를 실행하기에 충분하면서 Windows가 사용할 CPU도 남길 수 있다.
- **Memory 4 GB**: Ubuntu Server와 이후 설치할 Nginx, 모니터링 도구, 간단한 데이터베이스를 여유 있게 실행하기 위한 값이다.
- **Disk 30 GB**: 운영체제, 패키지, 로그, 실습 파일을 저장할 공간을 확보하되 Host 디스크를 과도하게 사용하지 않는 크기이다.
- **동적 할당**: 처음부터 Windows 디스크의 30 GB를 모두 차지하지 않고, VM에서 실제로 사용한 양에 따라 가상 디스크 파일이 커진다.
- **NAT**: 별도의 공유기 설정 없이 인터넷을 사용할 수 있고, VM을 외부 네트워크에 직접 노출하지 않아 초기 실습에 단순하고 안전하다.

VM에 CPU나 메모리를 너무 많이 할당하면 Windows가 느려질 수 있다. 반대로 너무 적게 할당하면 Ubuntu의 설치나 서비스 실행이 느려질 수 있으므로 Host와 Guest가 함께 사용할 수 있도록 균형을 잡아야 한다.

## Ubuntu Server LTS를 선택한 이유

Ubuntu Desktop 대신 Server 버전을 선택한 이유는 실제 서버처럼 GUI에 의존하지 않고 CLI로 운영하는 연습을 하기 위해서다. Server 버전은 기본 설치가 비교적 가볍고, 서비스·네트워크·로그를 직접 확인하는 학습에 적합하다.

LTS는 Long Term Support의 약자로 장기간 보안 업데이트와 유지보수를 제공하는 버전이다. 운영 환경에서는 새 기능이 빠르게 추가되는 버전보다 안정적으로 지원되는 LTS를 사용하는 경우가 많아 실습 환경에도 적합하다.

## Ubuntu Server 설치

VirtualBox의 광학 드라이브에 Ubuntu Server LTS ISO를 연결하고 VM을 부팅했다. 설치 과정에서는 불필요한 설정을 늘리지 않고 기본값을 중심으로 진행했다.

- 언어: English
- 키보드: English (US)
- 네트워크: DHCP 자동 설정
- Proxy: 사용하지 않음
- Mirror: 기본값
- Storage: 자동 파티션과 기본 LVM 구성
- 추가 패키지: 선택하지 않음
- OpenSSH Server: 설치

처음에는 DHCP를 사용해 VirtualBox가 IP 주소를 자동으로 할당하게 했다. 고정 IP 설정까지 동시에 진행하면 설치 문제와 네트워크 설정 문제를 구분하기 어려우므로, 먼저 자동 설정으로 정상 통신을 확인한 뒤 네트워크 학습 단계에서 고정 IP를 다루기로 했다.

OpenSSH Server는 Windows에서 Ubuntu의 터미널에 원격 접속하기 위해 설치했다. 가상 머신 화면에서 직접 명령을 입력할 수도 있지만, SSH를 사용하면 실제 원격 Linux 서버에 접속하는 방식으로 실습할 수 있다.

실습 계정과 서버 이름은 다음과 같이 구성했다.

```text
username: linuxuser
hostname: linux-lab
```

일반 사용자로 로그인하고, 관리자 권한이 필요한 작업에서만 `sudo`를 사용하도록 했다.

## 설치 중 발생한 문제

처음 VM을 실행했을 때 부팅 가능한 운영체제가 없다는 화면이 나타났다. 원인은 Ubuntu Server ISO가 가상 광학 드라이브에 연결되지 않은 것이었다.

VirtualBox에서 다음 위치에 ISO를 연결한 뒤 다시 부팅했다.

```text
VM 설정 → 저장소 → 광학 드라이브 → Ubuntu Server ISO 선택
```

설치 실패가 아니라 VM에 설치 디스크가 연결되지 않은 상태였으며, ISO를 마운트한 뒤 정상적으로 설치를 진행했다.

## 설치 결과 확인

Ubuntu Server 로그인 후 다음 명령을 실행했다.

```bash
whoami
hostname
pwd
ip addr
```

각 명령으로 다음을 확인했다.

- `whoami`: `linuxuser` 계정으로 로그인했는지 확인
- `hostname`: 현재 서버 이름이 `linux-lab`인지 확인
- `pwd`: 로그인 시작 위치가 홈 디렉터리인지 확인
- `ip addr`: 네트워크 인터페이스에 IP 주소가 할당됐는지 확인

## SSH Server 확인

설치 과정에서 선택한 OpenSSH Server가 실행 중인지 확인했다.

```bash
sudo systemctl status ssh
```

출력에서 아래 상태를 확인했다.

```text
Active: active (running)
```

SSH 서버는 Ubuntu의 `22`번 포트에서 원격 접속 요청을 기다린다.

## NAT와 포트 포워딩 설정

NAT 환경에서 Ubuntu VM은 Windows를 통해 외부 네트워크에 연결할 수 있다. 반대로 Windows에서 VM의 SSH 서버로 접속하기 위해 VirtualBox에 포트 포워딩 규칙을 추가했다.

NAT는 내부의 Ubuntu 주소와 외부 통신에 사용하는 Windows 쪽 주소를 변환한다. VM에서 인터넷으로 나가는 연결은 쉽게 만들 수 있지만, 외부에서 VM으로 먼저 들어오는 연결은 기본적으로 제한된다. 따라서 SSH 요청만 명시적으로 전달하도록 포트 포워딩을 설정했다.

```text
VM 설정 → 네트워크 → 어댑터 1 → NAT
→ 고급 → 포트 포워딩
```

| 항목 | 설정값 |
| --- | --- |
| 이름 | SSH |
| 프로토콜 | TCP |
| 호스트 IP | 127.0.0.1 |
| 호스트 포트 | 2222 |
| 게스트 IP | 비워둠 |
| 게스트 포트 | 22 |

호스트 포트에 `22`가 아닌 `2222`를 사용한 이유는 Windows에서 이미 사용 중인 포트와의 충돌을 피하고 Host 포트와 Guest 포트가 서로 달라도 연결할 수 있음을 확인하기 위해서다. VM을 추가하면 `2223 → VM2:22`처럼 서로 다른 Host 포트로 구분할 수도 있다.

`127.0.0.1`은 현재 컴퓨터 자신을 가리키는 Loopback 주소이다. 호스트 IP를 이 주소로 제한하면 같은 Windows PC에서만 해당 포트 포워딩 규칙을 이용할 수 있어 실습 환경을 외부 네트워크에 불필요하게 노출하지 않는다.

연결 흐름은 다음과 같다.

```text
Windows 127.0.0.1:2222
        ↓
VirtualBox Port Forwarding
        ↓
Ubuntu Server:22
        ↓
OpenSSH Server
```

## Windows에서 SSH 접속

Windows PowerShell에서 다음 명령을 실행했다.

```powershell
ssh -p 2222 linuxuser@127.0.0.1
```

첫 접속에서 서버 지문 확인 메시지가 나오면 `yes`를 입력하고 Ubuntu 계정의 비밀번호로 인증했다. 접속 후 프롬프트가 다음과 같이 나타나는 것을 확인했다.

```text
linuxuser@linux-lab:~$
```

이로써 Windows에서 VirtualBox의 포트 포워딩을 거쳐 Ubuntu Server에 SSH로 접속하는 실습 환경을 완성했다.

## 배운 점

- VirtualBox의 Host는 Windows이고 Guest는 Ubuntu Server이다.
- Ubuntu Server ISO는 VM의 설치 디스크 역할을 한다.
- NAT는 VM이 Host를 통해 외부 네트워크를 사용하게 한다.
- 포트 포워딩은 Host의 특정 포트를 Guest의 서비스 포트로 연결한다.
- SSH 접속에는 서버 실행 여부, 포트, 사용자 계정, 연결 주소가 모두 맞아야 한다.
- 접속 문제가 생기면 VM 실행 상태부터 SSH 서비스와 포트 포워딩까지 연결 경로를 순서대로 확인해야 한다.

```text
VM 실행 확인
→ IP 할당 확인
→ SSH 서비스 확인
→ 포트 포워딩 확인
→ 접속 주소와 계정 확인
```
