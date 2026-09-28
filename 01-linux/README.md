# Linux Learning Log

> Linux 명령어를 외우는 데서 끝나지 않고, **왜 그렇게 동작하는지 이해하고 실제 Ubuntu Server에서 확인·문제 해결까지 할 수 있는 시스템 운영 역량**을 만들기 위한 학습 기록입니다.

## 🎯 학습 목표

Linux Master 2급은 빠뜨리지 않기 위한 최소 기준으로 사용하고, 실제 기록은 다음 수준을 목표로 합니다.

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

| 궁금한 내용 / 검색 키워드 | 문서 |
|---|---|
| VirtualBox, Ubuntu Server, VM, Host/Guest, NAT, Port Forwarding, SSH | [Day 0 — Ubuntu Server 실습 환경과 SSH](day-00-ubuntu-server-virtualbox-network-ssh.md) |
| `pwd`, `ls`, `cd`, `mkdir`, `cp`, `mv`, `rm`, 파일시스템, 경로, `>`, `>>`, Pipe | [Day 1 — Linux CLI·파일시스템·리다이렉션](day-01-linux-cli-filesystem-redirection-pipe.md) |
| `find`, `grep`, `wc`, `sort`, `uniq`, `cut`, `awk` | [Day 2-1 — 파일 검색과 텍스트 처리](day-02-01-find-grep-sort-uniq-cut-awk.md) |
| stdin, stdout, stderr, FD 0/1/2, Pipe, Record, Field | [Day 2-2 — 표준 입출력과 텍스트 처리 원리](day-02-02-stdin-stdout-stderr-fd-pipe.md) |
| `sed`, `diff`, 설정 파일 수정, 백업, 검증 | [Day 2-3 — sed·diff와 안전한 설정 파일 수정](day-02-03-sed-diff-config-editing.md) |
| 사용자, 그룹, UID/GID, `sudo`, `su`, `/etc/passwd`, `getent` | [Day 3-1 — 사용자·그룹·sudo](day-03-01-users-groups-uid-gid-sudo-su.md) |
| 계정 잠금, `passwd -l`, `userdel`, Offboarding | [Day 3-2 — 계정 잠금·삭제·Offboarding](day-03-02-account-lock-userdel-offboarding.md) |
| 파일 권한, rwx, 644/640/755, `chmod` | [Day 4-1 — 파일 권한과 chmod](day-04-01-file-permissions-rwx-chmod.md) |
| 디렉터리 권한, traverse, 부모 경로, symbolic chmod | [Day 4-2 — 디렉터리 권한과 경로 접근](day-04-02-directory-permissions-traverse-symbolic-chmod.md) |
| `chown`, `chgrp`, 소유권, 그룹, `getent` | [Day 4-3 — 소유권·그룹·chown/chgrp](day-04-03-ownership-chown-chgrp-getent.md) |
| SetUID, SetGID, Sticky Bit, 특수 권한 | [Day 4-4 — SetUID·SetGID·Sticky Bit](day-04-04-special-permissions-setuid-setgid-sticky.md) |
| Process, PID/PPID, `ps`, `pgrep`, `top`, Signal, `kill` | [Day 5-1 — 프로세스·Signal·kill](day-05-01-process-pid-ps-pgrep-top-signals-kill.md) |
| Job, `jobs`, `fg`, `bg`, Ctrl+Z, `nohup`, SIGHUP | [Day 5-2 — Job Control과 nohup](day-05-02-job-control-jobs-fg-bg-nohup.md) |
| systemd, Service, Daemon, PID 1, `systemctl`, active/enabled, cgroup | [Day 6-1 — systemd·서비스·systemctl](day-06-01-systemd-service-systemctl-status-cgroup.md) |
| journal, `journalctl`, `journald`, `-u`, `-f`, `-b`, `--since`, Priority | [Day 6-2 — journalctl과 서비스 로그 분석](day-06-02-journalctl-system-logs-filtering.md) |
| Unit 파일, `[Unit]`, `[Service]`, `[Install]`, `ExecStart`, `Requires`, `After`, enable, `daemon-reload` | [Day 6-3 — systemd Unit 파일과 서비스 시작 과정](day-06-03-unit-files-execstart-dependencies-enable-daemon-reload.md) |
| Unit 경로 우선순위, Drop-in Override, `systemctl edit`, `systemctl cat`, `systemctl show` | [Day 6-4 — Unit 확인·Override·cat/show](day-06-04-unit-override-systemctl-cat-show.md) |
| 직접 Service 만들기, `Type=simple`, Script, `chmod +x`, `start/stop`, `enable --now`, journal, symbolic link | [Day 6-5 — 직접 systemd Service 만들기와 Lifecycle 실습](day-06-05-custom-service-lifecycle-journal-enable.md) |
| `failed`, `ExecStart` 경로 오류, `Permission denied`, `reset-failed`, `Restart=on-failure`, systemd 장애 분석 | [Day 6-6 — systemd Service 장애 분석과 Troubleshooting](day-06-06-systemd-service-failure-troubleshooting.md) |
| Package, `.deb`, Dependency, Repository, 로컬 Package Index, `apt`, `dpkg`, `apt update/install/policy`, `dpkg -S/-L` | [Day 7-1 — Package·Repository·apt·dpkg 기초](day-07-01-package-apt-dpkg-repository-basics.md) |
| 지금까지 사용한 Linux 명령어와 Option 빠른 검색 | [Linux Command & Option Reference](reference-linux-commands-options.md) |

---

# 📚 공부 순서

파일 이름 앞 번호는 실제 학습 순서입니다.

| 순서 | 주제 | 핵심 내용 | 상태 |
|---:|---|---|---|
| 00 | [Ubuntu Server 실습 환경과 SSH](day-00-ubuntu-server-virtualbox-network-ssh.md) | VM, NAT, Port Forwarding, SSH | ✅ |
| 01 | [Linux CLI·파일시스템·리다이렉션](day-01-linux-cli-filesystem-redirection-pipe.md) | 기본 명령, 경로, 파일/디렉터리, Pipe | ✅ |
| 02-01 | [파일 검색과 텍스트 처리](day-02-01-find-grep-sort-uniq-cut-awk.md) | `find`, `grep`, `wc`, `sort`, `uniq`, `cut`, `awk` | ✅ |
| 02-02 | [표준 입출력과 텍스트 처리 원리](day-02-02-stdin-stdout-stderr-fd-pipe.md) | FD 0/1/2, stdin/stdout/stderr, Pipe | ✅ |
| 02-03 | [sed·diff와 안전한 설정 파일 수정](day-02-03-sed-diff-config-editing.md) | `sed`, `diff`, backup, 검증 | ✅ |
| 03-01 | [사용자·그룹·sudo](day-03-01-users-groups-uid-gid-sudo-su.md) | UID/GID, group, sudo, su, NSS | ✅ |
| 03-02 | [계정 잠금·삭제·Offboarding](day-03-02-account-lock-userdel-offboarding.md) | 계정 잠금/삭제, 잔여 UID/GID 파일 | ✅ |
| 04-01 | [파일 권한과 chmod](day-04-01-file-permissions-rwx-chmod.md) | rwx, 4/2/1, chmod | ✅ |
| 04-02 | [디렉터리 권한과 경로 접근](day-04-02-directory-permissions-traverse-symbolic-chmod.md) | directory rwx, traverse | ✅ |
| 04-03 | [소유권·그룹·chown/chgrp](day-04-03-ownership-chown-chgrp-getent.md) | owner/group, chown, chgrp, getent | ✅ |
| 04-04 | [SetUID·SetGID·Sticky Bit](day-04-04-special-permissions-setuid-setgid-sticky.md) | special permission | ✅ |
| 05-01 | [프로세스·Signal·kill](day-05-01-process-pid-ps-pgrep-top-signals-kill.md) | PID/PPID, ps, pgrep, top, Signal | ✅ |
| 05-02 | [Job Control과 nohup](day-05-02-job-control-jobs-fg-bg-nohup.md) | jobs, fg, bg, nohup, SIGHUP | ✅ |
| 06-01 | [systemd·서비스·systemctl](day-06-01-systemd-service-systemctl-status-cgroup.md) | Service/Daemon, PID 1, Unit, status, cgroup | ✅ |
| 06-02 | [journalctl과 서비스 로그 분석](day-06-02-journalctl-system-logs-filtering.md) | journald, log filter, Boot, Priority | ✅ |
| 06-03 | [systemd Unit 파일과 서비스 시작 과정](day-06-03-unit-files-execstart-dependencies-enable-daemon-reload.md) | Unit 섹션, Dependency/Ordering, ExecStart, enable | ✅ |
| 06-04 | [Unit 확인·Override·cat/show](day-06-04-unit-override-systemctl-cat-show.md) | Override, cat/show, Property, Unit file 조회 | ✅ |
| 06-05 | [직접 systemd Service 만들기와 Lifecycle 실습](day-06-05-custom-service-lifecycle-journal-enable.md) | Script/Unit 작성, daemon-reload, start/stop, journal, enable/disable, symbolic link | ✅ |
| 06-06 | [systemd Service 장애 분석과 Troubleshooting](day-06-06-systemd-service-failure-troubleshooting.md) | failed, ExecStart 오류, permission, daemon-reload 누락, Restart 정책, 복구 검증 | ✅ |
| 07-01 | [Package·Repository·apt·dpkg 기초](day-07-01-package-apt-dpkg-repository-basics.md) | Package/.deb, Dependency, Repository, 로컬 Index, apt/dpkg 역할, 기본 조회 명령 | ✅ |

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
→ systemctl show UNIT
→ Process
→ Port
→ Config / Permission / Dependency
→ 원인 판단
→ 필요한 조치
→ status / journal 재확인
→ 실제 기능 검증
```

## Unit 파일 변경

```text
현재 Unit 확인
→ 기본 Unit / Drop-in Override 확인
→ 변경
→ systemctl daemon-reload
→ systemctl show로 인식값 확인
→ 영향 판단
→ 필요하면 reload/restart
→ status
→ journal
→ 실제 기능 검증
```

## 직접 만든 Service 점검

```text
Script 존재/권한 확인
→ systemctl cat UNIT
→ ExecStart 경로 확인
→ daemon-reload 여부 확인
→ start
→ status
→ pgrep으로 Process 확인
→ journal 확인
→ 실제 동작 검증
```

---

# 🗺️ 다음 로드맵

| Day | 예정 주제 | 핵심 내용 |
|---|---|---|
| Day 7 | 패키지 관리 | `apt`, `dpkg`, Repository, 설치/업데이트/삭제 |
| Day 8 | 디스크 / 파일시스템 | `lsblk`, `df`, `du`, mount, filesystem, inode, LVM 기초 |
| Day 9 | 네트워크 | IP, Subnet, Gateway, DNS, Port, `ip`, `ping`, `ss`, `curl`, `dig` |
| Day 10 | SSH 운영 | SSH Key, `authorized_keys`, `sshd_config`, 접속 장애 분석 |
| Day 11 | 로그 / cron / 시스템 점검 | `/var/log`, cron, CPU/Memory/Disk 점검, Bash 기초 |
| Day 12 | 종합 Troubleshooting | Process·Service·Port·Log·Disk를 연결한 장애 대응 미니 프로젝트 |

---

## ✅ 현재 진행 상태

**Day 0 ~ Day 7-1 완료.**

Day 6 systemd / Service 관리 파트를 마치고, Day 7 패키지 관리 학습을 시작했습니다.

Day 7-1에서는 다음 핵심 개념을 정리했습니다.

```text
Package
→ 프로그램을 설치/업데이트/삭제할 수 있도록 묶은 배포 단위

Repository
→ 실제 Package와 Metadata를 제공하는 저장소

apt update
→ Repository의 Package 목록/Metadata를 로컬에 갱신

apt install
→ 로컬 Metadata를 참고해 실제 Package와 Dependency를 Repository에서 다운로드하고 설치

apt
→ Repository와 Dependency를 포함한 고수준 Package 관리

dpkg
→ Debian .deb Package와 로컬 설치 상태를 직접 다루는 저수준 도구
```

특히 다음 오해를 구분했습니다.

```text
Repository에 있는 모든 Package가 로컬에 미리 존재하는 것은 아님
→ 로컬에는 주로 Package Index / Metadata가 존재

apt update
≠ 설치된 프로그램 업데이트

apt --version
→ apt 프로그램 자체의 Version
```

다음 학습은 **Day 7-2 — APT Repository 구조와 apt update 내부 흐름**입니다.
