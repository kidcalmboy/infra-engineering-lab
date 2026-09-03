# Linux

Linux 시스템 운영에 필요한 기본 개념과 실습을 기록합니다.

## 학습 주제

- 파일 시스템과 기본 명령어
- 사용자, 그룹, 소유권과 권한
- 프로세스와 서비스(systemd)
- 디스크와 파일 시스템 관리
- 네트워크 설정과 점검
- SSH 원격 접속
- cron 작업 스케줄링
- 시스템 로그 분석
- 방화벽 기본 설정

각 기록에는 실습 환경, 명령어, 실행 결과, 문제 해결 과정과 배운 점을 포함합니다.

## 학습 기록

### Day 0 - Linux 실습 환경 구축

- [Day 0 - VirtualBox와 Ubuntu Server 실습 환경 구축](day-00-virtualbox-ubuntu-server-setup.md)
  - VirtualBox, Host/Guest OS, Ubuntu Server, NAT, 포트 포워딩, SSH 접속 환경 구축

### Day 1 - Linux 기본 CLI와 파일시스템

- [Day 1 - Linux 기본 명령어와 파일·디렉터리 조작](day-01-linux-basic-cli.md)
  - `pwd`, `ls`, `cd`, `mkdir`, `touch`, `cp`, `mv`, `rm`
  - 절대경로/상대경로, `cat`, `less`, `head`, `tail`, 리다이렉션, 파이프
  - `/etc`, `/var/log`, `/home`, `/tmp`, `/proc`, `/dev`, `/usr` 등 Linux 파일시스템 구조

### Day 2 - 검색과 텍스트 처리

- [Day 2 - 파일 검색과 텍스트 처리 기초](day-02-search-and-text-processing.md)
  - `find`, `grep`, `wc`, `sort`, `uniq`, `cut`, `awk`
  - 로그 검색, 필터링, 집계, 필드 기반 분석

- [Day 2 - 텍스트 처리 개념 심화](day-02-text-processing-concepts.md)
  - 표준 입력/출력, 파이프라인 데이터 흐름
  - `grep | cut | sort | uniq` 조합을 구조적으로 이해하는 방법
  - 로그 분석 시 각 명령어의 역할과 선택 기준

- [Day 2 - sed와 설정 파일 수정 실습](day-02-sed-and-config-editing.md)
  - `sed` Stream Editor 구조
  - 치환, 선택 출력, 삭제, `-i` 원본 수정
  - 설정 변경 시 백업 → 수정 → `diff` → 검증 흐름

### Day 3 - 사용자와 그룹 관리

- [Day 3 - 사용자, 그룹, sudo 기초](day-03-users-groups-sudo.md)
  - User/Group, UID/GID, `/etc/passwd`, `/etc/shadow`, `/etc/group`
  - `adduser`, `passwd`, `groupadd`, `usermod -aG`
  - `sudo`, `su`, `su -`, 로그인 세션과 그룹 반영

- [Day 3 - 계정 잠금과 사용자 삭제](day-03-account-lock-and-deletion.md)
  - `passwd -l`, `passwd -u`, `passwd -S`
  - 계정 잠금 상태 `P`, `L`, `NP`
  - `userdel`과 `userdel -r`, 홈 디렉터리와 UID/GID 잔여 파일
  - 안전한 계정 정리와 Troubleshooting 절차

### Day 4 - 파일 권한과 소유권

- [Day 4-1 - Linux 파일 권한 기초와 chmod](day-04-file-permissions-basics.md)
  - Owner / Group / Others
  - 파일의 `r`, `w`, `x` 권한
  - `ls -l`, `ls -ld` 출력 해석
  - 숫자 권한 `r=4`, `w=2`, `x=1`
  - `chmod 600`, `640`, `644`, `700`, `750`, `755`, `777`
  - 과도한 권한 설정과 최소 권한 원칙

> 이후 Day 4에서는 디렉터리의 `rwx`, 문자 방식 `chmod`, `chown`, `chgrp`를 이어서 학습합니다.

## 학습 방향

단순 명령어 암기보다 다음 흐름을 기준으로 학습합니다.

```text
개념 이해
→ 명령어 구조 이해
→ 직접 실습
→ 출력 결과 해석
→ 문제 상황 재현
→ 원인 분석
→ 해결
→ 검증
→ GitHub 기록
```

서버 운영 관점에서는 명령을 실행하는 것보다 **현재 상태를 먼저 확인하고, 변경 이유와 결과를 설명할 수 있는 것**을 중요하게 생각합니다.
