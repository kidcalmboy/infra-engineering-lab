# Linux Learning Log

> Linux 시스템 운영에 필요한 개념과 명령어를 직접 실습하고, 장애 상황에서 어떻게 확인하고 복구하는지 기록하는 학습 인덱스입니다.

## 📌 학습 방향

- 명령어 암기보다 **왜 사용하는지** 이해한다.
- 개념 → 명령어 → 실습 → 검증 → Troubleshooting 순서로 학습한다.
- 실제 서버 운영·SM·인프라 업무에서 어떻게 활용되는지 연결한다.
- 잘못 이해한 내용은 그대로 남기지 않고 정확한 개념으로 수정한다.
- 실습 중 발생한 문제와 해결 과정은 재현 가능하도록 기록한다.
- 실습과 Troubleshooting 문제는 **직접 Linux 환경에서 명령어를 실행하고 결과를 확인해 결론을 도출하는 방식**으로 진행한다.

## 📚 학습 기록

| Day | 주제 | 핵심 내용 |
|---|---|---|
| [Day 0](day-00-virtualbox-ubuntu-server-setup.md) | VirtualBox와 Ubuntu Server 환경 구축 | Host/Guest/VM, Ubuntu Server, NAT, 포트 포워딩, SSH |
| [Day 1](day-01-linux-basic-cli.md) | Linux 기본 CLI와 파일시스템 | `pwd`, `ls`, `cd`, `cp`, `mv`, `rm`, 경로, 리다이렉션, 파이프, 핵심 디렉터리 |
| [Day 2-1](day-02-search-and-text-processing.md) | 파일 검색과 텍스트 처리 실습 | `find`, `grep`, `wc`, `sort`, `uniq`, `cut`, `awk`, 로그 분석 |
| [Day 2-2](day-02-text-processing-concepts.md) | 텍스트 처리 개념 | 표준 입출력, 파이프라인, 필터링·추출·집계 흐름 |
| [Day 2-3](day-02-sed-and-config-editing.md) | `sed`와 설정 파일 수정 | 치환, `-i`, `diff`, 백업, 설정 변경 검증 |
| [Day 3-1](day-03-users-groups-sudo.md) | 사용자·그룹·sudo | UID/GID, `/etc/passwd`, `/etc/shadow`, `/etc/group`, `adduser`, `groupadd`, `usermod`, `su`, `sudo` |
| [Day 3-2](day-03-account-lock-and-deletion.md) | 계정 잠금과 삭제 | `passwd -l/-u/-S`, `userdel`, `userdel -r`, UID/GID 잔여 파일 |
| [Day 4-1](day-04-file-permissions-basics.md) | 파일 권한 기초 | Owner/Group/Others, `rwx`, 숫자 권한, `chmod`, `ls -l`, `ls -ld` |
| [Day 4-2](day-04-directory-permissions-and-symbolic-chmod.md) | 디렉터리 권한과 문자 방식 `chmod` | 파일/디렉터리 `rwx` 차이, 경로 접근, `u/g/o/a`, `+/-/=`, `Permission denied` 분석 |
| [Day 4-3](day-04-ownership-groups-and-getent.md) | 파일 소유권, 그룹과 `getent` | `chown`, `chgrp`, Primary/Supplementary Group, `id`, `groups`, `getent`, 그룹 기반 권한 관리 |
| [Day 4-4](day-04-special-permissions-and-sudo.md) | 특수 권한과 `sudo` | SetUID, SetGID, Sticky Bit, `/usr/bin/passwd`, 절대/상대 경로, `sudo` 판단, 공유 디렉터리 실습 |

## 🧭 전체 학습 로드맵

- 파일시스템과 기본 명령어
- 사용자·그룹·소유권·권한
- 프로세스와 서비스(systemd)
- 디스크와 파일시스템 관리
- 네트워크 설정과 점검
- SSH 원격 접속
- cron 작업 스케줄링
- 시스템 로그 분석
- 방화벽 기본 설정
- 서버 장애 대응과 Troubleshooting

## 📝 문서 작성 기준

각 학습 문서는 가능한 범위에서 다음 구조를 사용합니다.

```text
이번에 배운 내용
→ 목차
→ 명령어 빠른 복습
→ 개념
→ 상세 설명
→ 실습
→ 헷갈리기 쉬운 부분
→ Troubleshooting
→ 실무 포인트
→ 핵심 정리
→ 복습 문제
```

내용이 없는 섹션을 억지로 만들지는 않되, **개념·실습·운영 관점·복습 가능성**을 우선합니다.
