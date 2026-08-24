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
