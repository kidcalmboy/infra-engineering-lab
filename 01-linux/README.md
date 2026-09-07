# Linux Learning Log

> Linux 명령어를 외우는 데서 끝나지 않고, **왜 그렇게 동작하는지 이해하고 실제 Ubuntu Server에서 확인·문제 해결까지 할 수 있는 시스템 운영 역량**을 만들기 위한 학습 기록입니다.

## 🎯 학습 목표

이 기록은 Linux Master 2급 수준을 최종 목표로 삼지 않습니다.

**Linux Master 2급 개념은 빠뜨리지 않기 위한 최소 기준**으로 사용하고, 실제 기록은 다음 수준을 목표로 합니다.

```text
기본 개념 이해
→ 명령어와 옵션의 의미 이해
→ Linux 내부 동작과 연결
→ 실제 Ubuntu Server 실습
→ 출력 결과 해석
→ 장애 원인 추론
→ 안전한 조치
→ 결과 검증
→ 실무 운영 관점으로 설명
```

최종적으로는 주니어 **Linux / System / Infrastructure / Server Operation / SM 엔지니어**가 면접이나 실무에서 "왜 이 명령을 사용했는지" 설명할 수 있는 수준을 목표로 합니다.

---

## 🧭 학습 원칙

### 1. 처음 보는 사람도 이해할 수 있게 기록

용어나 기호를 처음 사용할 때 의미를 생략하지 않습니다.

예를 들어 권한을 설명한다면 단순히 `chmod 640`만 기록하지 않고 다음처럼 연결합니다.

```text
r = read    = 4
w = write   = 2
x = execute = 1

rw- = 4 + 2 = 6
r-- = 4     = 4
--- = 0     = 0

640
→ Owner  = rw-
→ Group  = r--
→ Others = ---
```

필요하면 왜 `4`, `2`, `1`을 사용하는지도 bit 표현과 연결해 설명합니다.

### 2. 명령어보다 개념을 먼저 이해

```text
개념
→ 왜 필요한가
→ 실제 동작 원리
→ 명령어
→ 옵션 / 인자
→ 출력 해석
→ 직접 실습
→ 검증
```

순서로 학습합니다.

### 3. 실행 성공과 목표 달성을 구분

명령어가 오류 없이 끝났다고 해서 원하는 상태가 만들어졌다는 뜻은 아닙니다.

```text
명령 실행
→ 결과 확인
→ 실제 목표 상태와 비교
→ 필요하면 추가 점검
```

예를 들어 설정 파일 변경 후에는 파일만 확인하지 않고 서비스 상태와 로그까지 검증하는 습관을 목표로 합니다.

### 4. Troubleshooting은 정답 명령어부터 외우지 않음

```text
증상
→ 가설
→ 확인 명령
→ 출력 해석
→ 원인 판단
→ 조치
→ 재검증
→ 재발 방지
```

순서로 접근합니다.

---

## 📝 학습 문서 작성 기준

각 Day 문서는 가능한 한 다음 구조를 사용합니다.

```text
# 주제

> 오늘 배운 내용 요약

## 📌 이번에 배운 내용
## 📚 목차
## ⚡ 명령어 빠른 복습

## 1. 핵심 개념
### 한 줄 정의
### 상세 설명
### 왜 필요한가
### 실제 동작 원리
### 예시
### 실무 관점

## 2. 주요 명령어
### 명령어 이름 / 약어 의미
### 기본 동작
### 문법
### 사용한 옵션
### 자주 쓰는 관련 옵션
### Argument 의미
### 주요 출력 컬럼 해석
### 주의점
### 검증 방법

## 3. 개념 비교
## 4. 🧪 실제 Ubuntu 실습
## 5. ⚠️ 헷갈리기 쉬운 부분
## 6. 🔧 Troubleshooting
## 7. 💼 실무 포인트
## 8. ✅ 핵심 정리
## 9. 🧠 복습 문제
```

내용이 없는 섹션을 억지로 만들지는 않지만, **개념 설명·옵션·출력 해석·실습·Troubleshooting·실무 활용·검증**은 가능한 한 빠뜨리지 않습니다.

---

## ⚙️ 명령어 정리 기준

새로운 명령어가 나오면 가능한 범위에서 아래 내용을 기록합니다.

| 항목 | 기록 기준 |
|---|---|
| 명령어 이름 | 이름 또는 약어가 무엇을 의미하는지 |
| 기본 동작 | 옵션 없이 실행하면 무엇을 하는지 |
| Option | 실제 사용한 옵션의 의미와 동작 변화 |
| 관련 Option | 실무에서 자주 쓰거나 함께 알아야 하는 옵션 |
| Argument | 파일명, PID, 사용자명 등 인자가 무엇을 의미하는지 |
| Output | 중요한 컬럼과 상태 값 해석 |
| Why | 왜 해당 명령을 선택했는지 |
| Practice | 실제 Ubuntu Server에서 실행한 내용 |
| Risk | 삭제·권한 변경·서비스 중단 등 위험 요소 |
| Verify | 실행 후 원하는 상태가 만들어졌는지 확인하는 방법 |
| Operations | 서버 운영에서 언제 사용하는지 |

예를 들어:

```bash
ps -fp 2075
```

이라면 단순히 "프로세스 확인"으로 끝내지 않고 다음을 설명하는 방식입니다.

```text
ps → Process Status
-f → full-format listing
-p → PID를 기준으로 대상 process 선택
2075 → 확인할 PID

주요 확인 항목
→ USER / PID / PPID / CMD

사용 시점
→ 프로세스를 종료하거나 조치하기 전에 대상 identity 재검증
```

---

## 🖥️ 실습 환경

```text
Host OS : Windows
Hypervisor : VirtualBox
Guest OS : Ubuntu Server
접속 방식 : Windows Terminal / PowerShell → SSH
```

현재 기본 SSH 실습 구조:

```text
Windows 127.0.0.1:2222
        ↓
VirtualBox NAT Port Forwarding
        ↓
Ubuntu Server :22
        ↓
OpenSSH Server
```

접속 예:

```bash
ssh -p 2222 linuxuser@127.0.0.1
```

---

## 📚 현재 GitHub에서 확인 가능한 학습 기록

| Day | 주제 | 핵심 내용 |
|---|---|---|
| [Day 0](day-00-virtualbox-ubuntu-server-setup.md) | VirtualBox와 Ubuntu Server 실습 환경 구축 | Host/Guest/VM, Hypervisor, Ubuntu Server, NAT, Port Forwarding, SSH |
| [Day 1](day-01-linux-basic-cli.md) | Linux 기본 CLI와 파일·디렉터리 조작 | `pwd`, `ls`, `cd`, 파일 조작, 경로, 출력 확인, 리다이렉션, Pipe, 주요 디렉터리 |

> 현재 GitHub의 `01-linux/` 디렉터리에서 실제로 확인되는 문서를 기준으로 링크를 유지합니다. 학습한 내용과 GitHub 파일 구성이 다르면 먼저 실제 저장소 상태를 확인한 뒤 인덱스를 갱신합니다.

---

## 🗺️ 전체 학습 로드맵

| 단계 | 주제 | 핵심 내용 |
|---|---|---|
| Day 0 | 실습 환경 구축 | VirtualBox, Ubuntu Server, NAT, Port Forwarding, SSH |
| Day 1 | 기본 CLI / 파일시스템 | 경로, 파일·디렉터리 조작, 표준 입출력, 리다이렉션, Pipe |
| Day 2 | 검색 / 텍스트 처리 | `find`, `grep`, `wc`, `sort`, `uniq`, `cut`, `awk`, `sed`, `diff` |
| Day 3 | 사용자 / 그룹 / 계정 | UID/GID, Primary/Supplementary Group, `sudo`, `su`, 계정 생성·잠금·삭제 |
| Day 4 | 권한 / 소유권 | Owner/Group/Others, `rwx`, `chmod`, `chown`, `chgrp`, SetUID, SetGID, Sticky Bit |
| Day 5 | 프로세스 / Job Control | Process, PID/PPID, `ps`, `pgrep`, `top`, Signal, `kill`, `jobs`, `bg`, `fg`, `nohup` |
| Day 6 | systemd / 서비스 관리 | Unit, Service, `systemctl`, `journalctl`, 서비스 장애 분석 |
| Day 7 | 패키지 관리 | `apt`, `dpkg`, Repository, 설치·업데이트·삭제 |
| Day 8 | 디스크 / 파일시스템 | `lsblk`, `df`, `du`, mount, filesystem, inode, LVM 기초 |
| Day 9 | 네트워크 | IP, Subnet, Gateway, DNS, Port, `ip`, `ping`, `ss`, `curl`, `dig` |
| Day 10 | SSH 운영 | SSH 인증, key pair, `authorized_keys`, `sshd_config`, 접속 장애 분석 |
| Day 11 | 로그 / cron / 시스템 점검 | `journalctl`, `/var/log`, cron, CPU/Memory/Disk 점검, Bash 기초 |
| Day 12 | 종합 Troubleshooting | 프로세스·서비스·포트·로그·디스크를 연결한 장애 대응 미니 프로젝트 |

---

## 💼 실무에서 목표로 하는 사고방식

### 파일·설정 변경

```text
현재 사용자/서버/경로 확인
→ 대상 확인
→ 권한/소유권 확인
→ 필요 시 백업
→ 변경
→ diff/내용 검증
→ 서비스 반영
→ 상태/로그 검증
```

### Permission denied

```text
현재 사용자 확인
→ UID/GID/Group 확인
→ 파일 Owner/Group 확인
→ 파일 permission 확인
→ 부모 Directory traverse 권한 확인
→ 필요한 최소 권한만 수정
→ 실제 접근 재검증
```

### 프로세스 장애

```text
대상 찾기
→ PID/PPID/USER/CMD 확인
→ 상태/CPU/Memory 확인
→ 영향 판단
→ SIGTERM
→ 종료 검증
→ 필요한 경우에만 강제 종료 검토
```

### 서비스 장애

```text
systemctl status
→ process 확인
→ port 확인
→ journal/log 확인
→ config 확인
→ 원인 판단
→ 조치
→ restart/reload
→ 상태·로그·실제 접속 검증
```

---

## ✅ 최종 목표

이 저장소의 목적은 Linux 명령어 개수를 많이 아는 것이 아닙니다.

최종 목표는 다음 질문에 스스로 답할 수 있는 것입니다.

```text
현재 서버에서 무슨 일이 일어나고 있는가?
왜 이런 결과가 나왔는가?
무엇을 확인해야 원인을 좁힐 수 있는가?
어떤 조치가 가장 안전한가?
조치 후 무엇으로 정상 상태를 검증할 것인가?
같은 문제가 다시 발생하지 않도록 무엇을 기록할 것인가?
```

**명령어를 사용하는 사람에서, 시스템 상태를 읽고 문제를 해결할 수 있는 운영 엔지니어로 성장하는 것**을 목표로 합니다.
