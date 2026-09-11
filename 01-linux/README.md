# Linux Learning Log

> Linux 명령어를 외우는 데서 끝나지 않고, **왜 그렇게 동작하는지 이해하고 실제 Ubuntu Server에서 확인·문제 해결까지 할 수 있는 시스템 운영 역량**을 만들기 위한 학습 기록입니다.

## 🎯 학습 목표

이 기록은 Linux Master 2급을 최종 목표로 삼지 않습니다. Linux Master 2급 개념은 빠뜨리지 않기 위한 최소 기준으로 사용하고, 실제 문서는 다음 수준을 목표로 합니다.

```text
개념 이해
→ 왜 필요한지 이해
→ 내부 동작과 연결
→ 명령어 / Option / Argument 이해
→ 실제 Ubuntu Server 실습
→ 출력 해석
→ 장애 원인 추론
→ 안전한 조치
→ 검증
→ 실무 운영 관점으로 설명
```

최종적으로는 주니어 **Linux / System / Infrastructure / Server Operation / SM 엔지니어**가 처음 보는 문제도 상태·로그·Process·Port·설정을 연결해서 분석할 수 있는 수준을 목표로 합니다.

---

# 🔎 궁금한 내용 바로 찾기

궁금한 주제가 생겼을 때 파일명을 하나씩 열어보지 않아도 되도록 **개념과 명령어 기준 검색 인덱스**를 먼저 둡니다.

| 궁금한 내용 / 검색 키워드 | 문서 |
|---|---|
| VirtualBox, Ubuntu Server, VM, Host/Guest, NAT, Port Forwarding, SSH | [Day 0 — Ubuntu Server 실습 환경과 SSH](day-00-ubuntu-server-virtualbox-network-ssh.md) |
| `pwd`, `ls`, `cd`, `mkdir`, `cp`, `mv`, `rm`, 파일시스템, 경로, `>`, `>>`, Pipe | [Day 1 — Linux CLI·파일시스템·리다이렉션](day-01-linux-cli-filesystem-redirection-pipe.md) |
| `find`, `grep`, `wc`, `sort`, `uniq`, `cut`, `awk` | [Day 2-1 — 파일 검색과 텍스트 처리](day-02-01-find-grep-sort-uniq-cut-awk.md) |
| stdin, stdout, stderr, FD 0/1/2, Pipe, Record, Field | [Day 2-2 — 표준 입출력과 텍스트 처리 원리](day-02-02-stdin-stdout-stderr-fd-pipe.md) |
| `sed`, `diff`, 설정 파일 수정, 백업, 검증 | [Day 2-3 — sed·diff와 안전한 설정 파일 수정](day-02-03-sed-diff-config-editing.md) |
| 사용자, 그룹, UID/GID, `sudo`, `su`, `/etc/passwd`, `getent` | [Day 3-1 — 사용자·그룹·sudo](day-03-01-users-groups-uid-gid-sudo-su.md) |
| 계정 잠금, `passwd -l`, `userdel`, 퇴사자/계정 Offboarding | [Day 3-2 — 계정 잠금·삭제·Offboarding](day-03-02-account-lock-userdel-offboarding.md) |
| 파일 권한, Owner/Group/Others, rwx, 644/640/755, `chmod` | [Day 4-1 — 파일 권한과 chmod](day-04-01-file-permissions-rwx-chmod.md) |
| 디렉터리 권한, traverse, 부모 경로, symbolic chmod | [Day 4-2 — 디렉터리 권한과 경로 접근](day-04-02-directory-permissions-traverse-symbolic-chmod.md) |
| `chown`, `chgrp`, 소유권, 그룹, `getent` | [Day 4-3 — 소유권·그룹·chown/chgrp](day-04-03-ownership-chown-chgrp-getent.md) |
| SetUID, SetGID, Sticky Bit, 특수 권한, 공유 디렉터리 | [Day 4-4 — SetUID·SetGID·Sticky Bit](day-04-04-special-permissions-setuid-setgid-sticky.md) |
| Process, PID/PPID, `ps`, `pgrep`, `top`, Signal, `kill` | [Day 5-1 — 프로세스·Signal·kill](day-05-01-process-pid-ps-pgrep-top-signals-kill.md) |
| Job, `jobs`, `fg`, `bg`, Ctrl+Z, `nohup`, SIGHUP | [Day 5-2 — Job Control과 nohup](day-05-02-job-control-jobs-fg-bg-nohup.md) |
| systemd, Service, Daemon, PID 1, `systemctl`, active/enabled, cgroup | [Day 6-1 — systemd·서비스·systemctl](day-06-01-systemd-service-systemctl-status-cgroup.md) |
| journal, `journalctl`, `journald`, `-u`, `-f`, `-b`, `--since`, Priority | [Day 6-2 — journalctl과 서비스 로그 분석](day-06-02-journalctl-system-logs-filtering.md) |
| Unit 파일, `[Unit]`, `[Service]`, `[Install]`, `ExecStart`, `Requires`, `After`, enable, `daemon-reload` | [Day 6-3 — systemd Unit 파일과 서비스 시작 과정](day-06-03-unit-files-execstart-dependencies-enable-daemon-reload.md) |
| 지금까지 사용한 Linux 명령어와 Option 빠른 검색 | [Linux Command & Option Reference](reference-linux-commands-options.md) |

---

# 📚 공부 순서

파일 이름 앞의 번호는 **실제로 공부한 순서**를 나타냅니다. 같은 Day 안에서는 `01`, `02`, `03` 순서로 이어집니다.

| 순서 | 주제 | 핵심 내용 | 상태 |
|---:|---|---|---|
| 00 | [Ubuntu Server 실습 환경과 SSH](day-00-ubuntu-server-virtualbox-network-ssh.md) | VM, Host/Guest, Linux/Ubuntu, NAT, Port Forwarding, SSH | ✅ |
| 01 | [Linux CLI·파일시스템·리다이렉션](day-01-linux-cli-filesystem-redirection-pipe.md) | 기본 명령어, 경로, 파일/디렉터리, stdin/stdout 기초, Pipe | ✅ |
| 02-01 | [파일 검색과 텍스트 처리](day-02-01-find-grep-sort-uniq-cut-awk.md) | `find`, `grep`, `wc`, `sort`, `uniq`, `cut`, `awk` | ✅ |
| 02-02 | [표준 입출력과 텍스트 처리 원리](day-02-02-stdin-stdout-stderr-fd-pipe.md) | FD 0/1/2, stdin/stdout/stderr, Pipe, Record/Field | ✅ |
| 02-03 | [sed·diff와 안전한 설정 파일 수정](day-02-03-sed-diff-config-editing.md) | `sed`, `diff`, preview, backup, 검증 | ✅ |
| 03-01 | [사용자·그룹·sudo](day-03-01-users-groups-uid-gid-sudo-su.md) | UID/GID, primary/supplementary group, sudo, su, NSS | ✅ |
| 03-02 | [계정 잠금·삭제·Offboarding](day-03-02-account-lock-userdel-offboarding.md) | password lock, `userdel`, 잔여 UID/GID 파일, 운영 Offboarding | ✅ |
| 04-01 | [파일 권한과 chmod](day-04-01-file-permissions-rwx-chmod.md) | rwx, 4/2/1, 600/640/644/755, 최소 권한 | ✅ |
| 04-02 | [디렉터리 권한과 경로 접근](day-04-02-directory-permissions-traverse-symbolic-chmod.md) | directory rwx, traverse, parent permission, symbolic chmod | ✅ |
| 04-03 | [소유권·그룹·chown/chgrp](day-04-03-ownership-chown-chgrp-getent.md) | owner/group, UID/GID mapping, `chown`, `chgrp`, `getent` | ✅ |
| 04-04 | [SetUID·SetGID·Sticky Bit](day-04-04-special-permissions-setuid-setgid-sticky.md) | special permission, Effective UID/GID, 공유 디렉터리 | ✅ |
| 05-01 | [프로세스·Signal·kill](day-05-01-process-pid-ps-pgrep-top-signals-kill.md) | PID/PPID, State, `ps`, `pgrep`, `top`, SIGTERM/SIGKILL | ✅ |
| 05-02 | [Job Control과 nohup](day-05-02-job-control-jobs-fg-bg-nohup.md) | Job ID, foreground/background, `jobs`, `fg`, `bg`, SIGHUP | ✅ |
| 06-01 | [systemd·서비스·systemctl](day-06-01-systemd-service-systemctl-status-cgroup.md) | Service/Daemon, PID 1, Unit, oneshot, active/enabled, status, cgroup | ✅ |
| 06-02 | [journalctl과 서비스 로그 분석](day-06-02-journalctl-system-logs-filtering.md) | journald, 로그 필터, Boot, 시간 범위, Priority | ✅ |
| 06-03 | [systemd Unit 파일과 서비스 시작 과정](day-06-03-unit-files-execstart-dependencies-enable-daemon-reload.md) | Unit 섹션, Dependency/Ordering, ExecStart, Target, enable, daemon-reload | 🔄 |

---

# 🧭 학습 원칙

## 1. 개념부터 이해

```text
한 줄 정의
→ 왜 필요한가
→ 실제 동작 원리
→ 관련 용어
→ 개념 비교
→ 명령어
→ Option / Argument
→ 출력 해석
→ 직접 실습
→ 검증
→ Troubleshooting
→ 실무 관점
```

## 2. 명령 실행 성공과 목표 달성을 구분

```text
명령 실행 성공
≠
실제 서비스/시스템 정상
```

예를 들어 서비스가 `active`여도 Port, Firewall, DNS, Backend 등의 문제로 실제 기능은 실패할 수 있습니다.

## 3. Troubleshooting 순서

```text
증상
→ 가설
→ 확인
→ 출력 해석
→ 원인
→ 조치
→ 재검증
→ 실제 기능 검증
→ 재발 방지
```

## 4. 위험한 명령은 영향부터 확인

삭제, 권한 변경, Process 종료, Service restart 같은 명령은 다음 순서를 우선합니다.

```text
현재 상태 확인
→ 대상 식별
→ 영향 범위 판단
→ 최소 변경
→ 검증
```

---

# 🖥️ 실습 환경

```text
Host OS      : Windows
Hypervisor   : VirtualBox
Guest OS     : Ubuntu Server
Linux User   : linuxuser
접속 방식     : Windows Terminal / PowerShell → SSH
```

SSH 실습 구조:

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

# 💼 누적 운영 체크 흐름

## Permission denied

```text
whoami / id
→ 파일 Owner/Group
→ 파일 permission
→ 부모 Directory traverse permission
→ 필요한 최소 변경
→ 실제 접근 검증
```

## Process 이상

```text
pgrep으로 후보 찾기
→ ps로 PID/USER/PPID/CMD 확인
→ CPU/Memory/State 확인
→ 영향 판단
→ SIGTERM
→ 종료 검증
→ 필요한 경우에만 SIGKILL 검토
```

## Service 장애

```text
systemctl status UNIT
→ journalctl -u UNIT
→ systemctl cat UNIT
→ Process
→ Port
→ Config / Permission / Dependency
→ 원인 판단
→ reload/restart 등 필요한 조치
→ status / journal 재확인
→ 실제 기능 검증
```

## Unit 파일 변경

```text
현재 Unit 확인
→ 원본 / override 확인
→ 변경
→ systemctl daemon-reload
→ 영향 판단
→ 필요하면 reload/restart
→ status
→ journal
→ 실제 기능 검증
```

---

# 🗺️ 다음 로드맵

| Day | 예정 주제 | 핵심 내용 |
|---|---|---|
| Day 6 | systemd / 서비스 관리 마무리 | 안전한 연습용 Service Unit 생성, lifecycle 실습, 장애 분석 |
| Day 7 | 패키지 관리 | `apt`, `dpkg`, Repository, 설치/업데이트/삭제 |
| Day 8 | 디스크 / 파일시스템 | `lsblk`, `df`, `du`, mount, filesystem, inode, LVM 기초 |
| Day 9 | 네트워크 | IP, Subnet, Gateway, DNS, Port, `ip`, `ping`, `ss`, `curl`, `dig` |
| Day 10 | SSH 운영 | SSH Key, `authorized_keys`, `sshd_config`, 접속 장애 분석 |
| Day 11 | 로그 / cron / 시스템 점검 | `/var/log`, cron, CPU/Memory/Disk 점검, Bash 기초 |
| Day 12 | 종합 Troubleshooting | Process·Service·Port·Log·Disk를 연결한 장애 대응 미니 프로젝트 |

---

## ✅ 현재 진행 상태

**Day 0 ~ Day 6-2 완료, Day 6-3 진행 중.**

현재는 systemd 서비스가 실제 Unit 파일을 통해 어떻게 실행되고, Dependency와 Ordering이 어떻게 나뉘며, `enable`과 `daemon-reload`가 어떤 역할을 하는지까지 학습했습니다.

다음은 SSH를 건드리지 않고 `systemctl cat/show`로 실제 Unit을 관찰한 뒤, **안전한 연습용 Service Unit을 직접 만들어 lifecycle과 장애 분석을 실습**하는 단계입니다.
