# 1일 차 | 리눅스 서버 상태 점검과 systemd 서비스 장애 복구

> **핵심 요약**  
> Ubuntu Server에 SSH로 접속해 CPU·메모리·서비스 상태를 점검하고, 실습 서비스를 정상 중지·재시작한 뒤 SIGKILL로 비정상 종료를 재현했다. systemd의 `Restart=on-failure` 정책에 따른 자동 복구를 **실제 journal 로그와 PID 변경**으로 검증했다.

| 항목 | 내용 |
| :-- | :-- |
| 학습일 | 2026-11-12 |
| 분야 | Linux 서버 운영 · 서비스 관리 · 장애 대응 |
| 실습 환경 | Windows / VirtualBox `ubuntu-server-lab` / Ubuntu Server 26.04 LTS |
| 커널 | Linux `7.0.0-30-generic` · x86-64 |
| 접속 | PowerShell → SSH (`127.0.0.1:2222` 포트 포워딩) |
| 실습 대상 | `hello-systemd.service` (기존 학습용 서비스) |
| 최종 결과 | 서비스 정상 중지·수동 복구 및 비정상 종료 후 자동 복구 검증 |
| 이해도 평가 | **미완료** (3문항 미제출) |

---

## 목차

1. [학습 목표와 완료 기준](#1-학습-목표와-완료-기준)
2. [핵심 개념 정리](#2-핵심-개념-정리)
3. [실습 환경과 자원 점검](#3-실습-환경과-자원-점검)
4. [systemd 서비스 구조 분석](#4-systemd-서비스-구조-분석)
5. [실습 A — 정상 중지 및 수동 복구](#5-실습-a--정상-중지-및-수동-복구)
6. [실습 B — 비정상 종료와 자동 복구](#6-실습-b--비정상-종료와-자동-복구)
7. [명령어 치트시트](#7-명령어-치트시트)
8. [결과·회고·다음 학습](#8-결과회고다음-학습)

## 1. 학습 목표와 완료 기준

**목표:** 서버의 기초 상태를 읽고, 서비스 관리 명령의 의미를 이해하며, 장애 상황에서 **증상 → 원인 → 조치 → 검증** 순서로 근거를 남기는 것.

- [x] SSH로 실습용 Ubuntu Server 접속
- [x] CPU·메모리·Load Average 해석
- [x] 실행 중인 서비스와 Unit 설정 확인
- [x] 정상 중지와 수동 시작의 상태 차이 확인
- [x] SIGKILL 비정상 종료 재현
- [x] systemd 자동 재시작을 로그로 검증
- [ ] 개념·상황형 문제 3개 평가 (아직 미제출)

> **실습 원칙:** `ssh.service` 같은 접속 필수 서비스는 중지하지 않는다. 서비스의 `active` 상태만으로 기능 정상성을 확정하지 않으며, 실행 결과·PID·로그를 함께 확인한다.

## 2. 핵심 개념 정리

### 2-1. 서버 자원과 부하 지표

| 개념 | 정의 | 확인 방법 | 판단할 때 주의할 점 |
| :-- | :-- | :-- | :-- |
| 논리 CPU | OS가 작업 실행에 사용할 수 있는 논리 처리 단위 | `nproc` | 물리 CPU 코어 수와 동일하지 않을 수 있음 |
| Load Average | 일정 시간 동안 실행 중/CPU 실행 대기 및 일부 I/O 대기 작업의 평균 부하 | `uptime`, `top` | **CPU 사용률(%)이 아님**. 논리 CPU 수와 I/O 상태를 함께 봄 |
| CPU `id` | CPU가 유휴 상태인 시간 비율 | `top` | 높을수록 측정 구간에 CPU가 비교적 한가했음을 의미 |
| CPU `wa` | CPU의 I/O 대기 시간 비율 | `top` | 높게 지속되면 디스크 등 I/O 원인 조사 |
| 메모리 `free` | 사용되지 않는 메모리 | `top` | 캐시로 쓰는 메모리를 고려하지 않으면 오해하기 쉬움 |
| 메모리 `available` | 스왑을 크게 쓰지 않고 새 작업에 제공 가능할 것으로 추정되는 메모리 | `top` | 서버 메모리 여유 판단에 유용 |
| Swap | 메모리 압박 시 활용 가능한 디스크 기반 공간 | `top` | 사용량만으로 성능 문제를 확정할 수 없음 |

**예시:** 논리 CPU 2개에서 Load Average 6.0이 지속된다면 CPU 경합 또는 I/O 대기 원인을 추가 확인해야 한다. 이를 CPU 사용률 300%라고 해석하면 안 된다.

### 2-2. systemd · systemctl · Unit

| 용어 | 역할 | 예 |
| :-- | :-- | :-- |
| `systemd` | 서비스·시스템 상태를 관리하는 서비스 관리자 | 프로세스 시작·재시작 정책 적용 |
| `systemctl` | systemd에 조회·제어 명령을 보내는 도구 | `systemctl status ssh` |
| Unit 파일 | 서비스를 어떻게 실행하고 관리할지 정의한 설정 | `/etc/systemd/system/hello-systemd.service` |
| `journalctl` | systemd Journal에 기록된 로그를 조회 | `journalctl -u ...` |

**현재 실행 상태와 부팅 설정은 서로 다르다.**

| 상태 | 의미 |
| :-- | :-- |
| `active (running)` | 현재 서비스가 실행 중 |
| `inactive (dead)` | 현재 서비스가 중지됨 |
| `failed` | systemd가 서비스 실패 상태로 판단함 |
| `enabled` | 부팅 시 자동 시작하도록 설정됨 |
| `disabled` | 해당 자동 시작 연결이 설정되지 않음 |

따라서 **`enabled`이면서 `inactive (dead)`**일 수 있다. `systemctl stop`은 현재 서비스를 중지하고, `systemctl disable`은 부팅 자동 시작 설정을 해제한다.

### 2-3. 프로세스 종료와 재시작

- **SIGTERM:** 일반적인 종료 요청. 프로세스가 종료 정리 작업을 수행할 기회를 가질 수 있다.
- **SIGKILL:** 강제 종료 신호. 프로세스가 신호를 처리하거나 무시할 수 없다.
- **`Restart=on-failure`:** 비정상 종료 등 실패 조건에서 systemd가 자동 재시작을 시도하는 정책. 관리자에 의한 정상 `systemctl stop`은 보통 재시작 대상이 아니다.
- **PID:** 프로세스 식별 번호. 새 프로세스로 재시작되면 일반적으로 변경된다.
- **`--no-pager`:** 결과를 `less` 같은 pager로 나누지 않고 바로 출력. 서비스 상태는 바꾸지 않음.
- **`sudo`:** 허가된 명령을 다른 사용자(기본값 root)의 권한으로 실행. 무조건 모든 명령이 허용되는 것은 아니다.

## 3. 실습 환경과 자원 점검

### 3-1. SSH 접속

```powershell
ssh -p 2222 linuxuser@127.0.0.1
```

**관찰:** Windows PowerShell에서 SSH 접속 성공. `127.0.0.1`은 호스트의 루프백 주소이고 `2222`는 접속에 사용한 호스트 측 포트이다. VirtualBox NAT 포트 포워딩을 이용한 구성으로 추정된다.

### 3-2. 점검 명령과 실제 출력

```bash
hostnamectl
uptime
nproc
top
```

| 점검 지표 | 측정 결과 | 해석 |
| :-- | :-- | :-- |
| OS / 커널 | Ubuntu 26.04 LTS / Linux 7.0.0-30-generic | 환경 확인 |
| 가상화 | Oracle VirtualBox | 가상 서버 확인 |
| 가동 시간 | `up 5 min` (첫 `uptime` 측정) | 부팅 직후 |
| Load Average | `0.08, 0.08, 0.03` (첫 측정) | 낮은 부하 |
| 논리 CPU | `2` | CPU 수 확인 |
| `top` Load Average | `0.04, 0.05, 0.01` (이후 측정) | 낮은 부하 |
| CPU idle / I/O wait | `93.3% / 0.2%` | 해당 시점에 뚜렷한 병목 징후 없음 |
| CPU softirq | `5.9%` | 지속 시 인터럽트 부하 원인 분석 필요 |
| 프로세스 | 117 total / 1 running / 116 sleeping / 0 zombie | 좀비 작업 없음 |
| RAM | 3398.8 MiB total / 422.9 used / 2975.9 available | 메모리 여유 있음 |
| Swap | 2709.0 MiB total / 0 used | 해당 시점 미사용 |

> **판정:** 측정한 시점에는 눈에 띄는 CPU·메모리 부족 징후가 없다. 그러나 단일 스냅샷으로 서비스 응답성이나 장기 안정성을 보증할 수는 없다.

## 4. systemd 서비스 구조 분석

### 4-1. 실행 중인 서비스 확인

```bash
systemctl list-units --type=service --state=running --no-pager
systemctl status hello-systemd.service --no-pager
systemctl cat hello-systemd.service
```

**관찰:** 실행 중인 서비스 19개 확인. `ssh`, `cron`, `chrony`, `rsyslog`, `systemd-journald` 등과 함께 이전에 만든 `hello-systemd.service`가 실행 중이었다.

### 4-2. 실제 Unit 파일

**경로:** `/etc/systemd/system/hello-systemd.service`

```ini
[Unit]
Description=Hello Systemd Practice Service

[Service]
Type=simple
ExecStart=/usr/local/bin/hello-systemd.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

| 설정 | 의미 |
| :-- | :-- |
| `[Unit]` / `Description` | 서비스 설명 및 Unit 관련 정보 |
| `Type=simple` | 시작된 프로세스를 서비스의 주 프로세스로 관리 |
| `ExecStart` | 실행할 프로그램/스크립트 경로 |
| `Restart=on-failure` | 실패 조건에서 재시작 시도 |
| `[Install]` / `WantedBy=multi-user.target` | enable 시 연결할 시작 대상 |

### 4-3. 실제 실행 스크립트

**경로:** `/usr/local/bin/hello-systemd.sh`

```bash
#!/bin/bash

while true
do
    echo "hello from systemd"
    sleep 10
done
```

**동작:** Bash 반복문이 메시지를 출력하고 10초간 대기한다. 서비스 주 프로세스는 Bash이며 `sleep 10`은 하위 프로세스로 나타날 수 있다. 이 스크립트의 출력은 Journal에서 확인됐다.

## 5. 실습 A — 정상 중지 및 수동 복구

### 목표

운영자가 의도적으로 서비스를 중지했을 때의 상태 변화를 확인하고, 다시 시작한 뒤 실행·로그를 검증한다.

### 실행 절차

```bash
# 1. 서비스 중지
sudo systemctl stop hello-systemd.service

# 2. 중지 상태 확인
systemctl status hello-systemd.service --no-pager

# 3. 서비스 재시작
sudo systemctl start hello-systemd.service

# 4. 복구 상태 확인
systemctl status hello-systemd.service --no-pager
```

### 실제 검증 결과

| 단계 | 증거 | 판정 |
| :-- | :-- | :-- |
| 중지 | `Active: inactive (dead)` | 정상 중지 |
| 종료 처리 | `signal=TERM`, `Deactivated successfully` | systemd가 정상 종료로 처리 |
| 수동 시작 | `Active: active (running)` | 실행 상태 복귀 |
| 프로세스 확인 | PID `1006 → 1513` | 새 주 프로세스 확인 |
| 기능 확인 | `hello from systemd` 10초 간격 출력 | 반복 동작 재개 |

**결론:** 의도적인 정상 중지와 수동 복구를 완료했다. **이 단계는 비정상 장애가 아니다.**

## 6. 실습 B — 비정상 종료와 자동 복구

### 6-1. 장애 대응 기록

| 항목 | 내용 |
| :-- | :-- |
| 사례 ID | LAB-20261112-01 |
| 장애 종류 | 실습용 서비스 주 프로세스 강제 종료 |
| 장애 발생 시각 | 2026-11-12 **05:14:53 UTC** |
| 영향 범위 | `hello-systemd.service` 실습 프로세스 |
| 장애 주입 방법 | `SIGKILL` |
| 감지 | systemd의 실패 감지 |
| 복구 방식 | `Restart=on-failure` 정책에 따른 자동 재시작 |
| 결과 | 새 PID 1806으로 실행 및 출력 재개 |
| 서비스 영향 시간 | 초 단위 로그만 있어 정밀 측정 불가 |

### 6-2. 재현·검증 명령

> **주의:** 아래 SIGKILL 명령은 격리된 학습용 서비스에만 실행한다. 다른 운영 서비스에 적용하지 않는다.

```bash
# 1. 주 프로세스에 SIGKILL 전송 (장애 주입)
sudo systemctl kill --kill-whom=main --signal=SIGKILL hello-systemd.service

# 2. 현재 상태 확인
systemctl status hello-systemd.service --no-pager

# 3. 최근 장애·복구 로그 확인
journalctl -u hello-systemd.service -n 30 --no-pager
```

### 6-3. 실제 로그 증거

```text
Nov 12 05:14:53 ubuntu-server systemd[1]: hello-systemd.service: Sent signal SIGKILL to main process 1513 (hello-systemd.s) on client request.
Nov 12 05:14:53 ubuntu-server systemd[1]: hello-systemd.service: Main process exited, code=killed, status=9/KILL
Nov 12 05:14:53 ubuntu-server systemd[1]: hello-systemd.service: Failed with result 'signal'.
Nov 12 05:14:53 ubuntu-server systemd[1]: hello-systemd.service: Scheduled restart job, restart counter is at 1.
Nov 12 05:14:53 ubuntu-server systemd[1]: Started hello-systemd.service - Hello Systemd Practice Service.
Nov 12 05:14:53 ubuntu-server hello-systemd.sh[1806]: hello from systemd
Nov 12 05:15:03 ubuntu-server hello-systemd.sh[1806]: hello from systemd
```

### 6-4. 원인 → 탐지 → 복구 → 검증

| 순서 | 분석 | 근거 |
| :-- | :-- | :-- |
| ① 원인 | 주 프로세스 PID 1513 강제 종료 | `Sent signal SIGKILL` |
| ② 탐지 | systemd가 신호에 의한 종료를 실패로 판단 | `status=9/KILL`, `Failed with result 'signal'` |
| ③ 복구 | 재시작 작업 1회 예약 및 실행 | `Scheduled restart job`, `Started` |
| ④ 검증 | 새 PID 1806에서 메시지 출력 재개 | `hello-systemd.sh[1806]` |
| ⑤ 최종 판정 | **자동 복구 성공** | 실패 감지·재시작·동작 재개 로그 일치 |

**개선 아이디어:** 실제 운영 환경이라면 로그의 시작 성공만 보지 않고 애플리케이션 헬스체크, 알림, 재시작 빈도, 사용자 영향 및 재발 방지책도 추가한다.

> **사례의 한계:** 통제된 실습 장애다. 실제 고객 서비스 장애 해결 경험으로 표현하지 않는다.

## 7. 명령어 치트시트

| 명령어 | 실무 목적 |
| :-- | :-- |
| `hostnamectl` | 서버 OS·호스트명·커널 확인 |
| `uptime` | 가동 시간·Load Average 확인 |
| `nproc` | 사용 가능 논리 CPU 확인 |
| `top` | 실시간 자원·프로세스 관찰 (`q` 종료) |
| `systemctl list-units --type=service --state=running --no-pager` | 현재 실행 중인 서비스 목록 |
| `systemctl status hello-systemd.service --no-pager` | 서비스 상태·PID·최근 로그 조회 |
| `systemctl cat hello-systemd.service` | 적용된 Unit 정의 확인 |
| `cat /usr/local/bin/hello-systemd.sh` | 실습 스크립트 내용 조회 |
| `sudo systemctl stop hello-systemd.service` | 정상 중지 |
| `sudo systemctl start hello-systemd.service` | 서비스 시작 |
| `sudo systemctl kill --kill-whom=main --signal=SIGKILL hello-systemd.service` | **실습 전용** 비정상 종료 주입 |
| `journalctl -u hello-systemd.service -n 30 --no-pager` | 특정 서비스 최신 로그 30줄 확인 |

## 8. 결과·회고·다음 학습

### 오늘의 결과

- **확인한 것:** 서비스가 정상 중지·재시작되고, 비정상 종료 후 재시작 정책으로 자동 복구됨.
- **새로 익힌 것:** `systemd`와 `systemctl`의 차이, `enabled`와 `active`의 차이, SIGTERM과 SIGKILL, Journal 기반 복구 검증.
- **실무적으로 중요했던 점:** 명령어가 오류 없이 끝났다는 사실만으로 복구를 단정하지 않고, **상태 + PID + 로그**를 확인한 것.
- **아직 미검증:** 독립적인 실무 장애 진단 능력, 장기 안정성, Day 1 최종 이해도 평가 3문항.

### 면접 대비 복습 질문

1. `systemd`와 `systemctl`의 차이는 무엇인가?
2. `enabled`인데 `inactive (dead)`일 수 있는 이유는?
3. `Restart=on-failure`는 왜 정상적인 `systemctl stop`과 SIGKILL을 다르게 다루는가?
4. 서비스가 `active (running)`인데도 접속 장애가 발생할 수 있는 이유는?
5. 자동 복구를 입증하려면 어떤 로그와 지표를 제시할 것인가?

**평가 상태:** 위 질문은 복습용이며, 별도의 Day 1 객관식 3문항 평가는 아직 제출되지 않았다.

### 다음 수업: 2일 차

**주제:** `journalctl` 로그 필터링, 오류 메시지 분류, 서비스 장애 원인 추적.

**예정 실습:** 특정 시간 구간·서비스 단위 로그 조회 → 문제 증상 정리 → 원인 후보 수립 → 검증 명령 선택.

---

*문서 작성 기준: 실제 터미널 출력과 실습 결과만 완료로 인정. 향후 학습 문서도 **개념 → 실행 절차 → 실제 결과 → 원인·해석 → 검증 → 회고** 구조로 정리한다.*
