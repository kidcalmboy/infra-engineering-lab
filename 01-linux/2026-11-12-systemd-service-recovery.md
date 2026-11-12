# Linux 서버 운영 실습 — systemd 서비스 관리 및 자동 복구

- **실습일:** 2026-11-12
- **환경:** Windows 호스트 / VirtualBox `ubuntu-server-lab` / Ubuntu Server 26.04 LTS / Linux 7.0.0-30-generic (x86-64)
- **접속:** PowerShell → SSH, 로컬 포트 2222 포워딩 (`ssh -p 2222 linuxuser@127.0.0.1`)
- **목표:** 서버 상태 확인, systemd 서비스 운영, 의도적 중지와 비정상 종료 비교, 재시작 정책 및 로그 검증
- **검증 범위:** 사용자가 공유한 명령어 출력 및 실습 결과에 근거함. 별도의 인프라 부하 테스트는 수행하지 않음.

## 1. 서버 상태 점검

| 명령어 | 결과 | 해석 |
|---|---|---|
| `hostnamectl` | Ubuntu 26.04 LTS, Kernel 7.0.0-30-generic, VirtualBox | 가상 서버 OS와 커널 확인 |
| `uptime` | up 5 min, 1 user, load average: 0.08, 0.08, 0.03 | 초기 부하 낮음 |
| `nproc` | 2 | 사용 가능한 논리 CPU 2개 |
| `top` | 117 tasks (1 running), CPU idle 93.3%, I/O wait 0.2% | 측정 시점에 CPU 경합 징후 낮음 |
| `top` | Mem 3398.8 MiB total, 422.9 MiB used, 2975.9 MiB available | 메모리 여유 있음 |
| `top` | Swap 2709.0 MiB total, 0.0 MiB used | Swap 미사용 |

**해석 시 주의:** Load Average는 CPU 사용률이 아니며 CPU 실행 대기 및 일부 I/O 대기 작업의 평균 부하를 반영한다. `top` 수치는 일시점 스냅샷이므로 서비스 SLA 또는 장기 안정성을 보장하지 않는다.

## 2. systemd 서비스 조사

```bash
systemctl list-units --type=service --state=running --no-pager
systemctl status hello-systemd.service --no-pager
systemctl cat hello-systemd.service
```

- 조회 당시 실행 중인 서비스는 19개.
- `ssh.service`, `cron.service`, `chrony.service`, `rsyslog.service`, `systemd-journald.service` 등이 실행 중.
- 기존 실습 서비스 `hello-systemd.service` 발견.
- Unit 설정:

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

- 실제 스크립트:

```bash
#!/bin/bash

while true
do
        echo "hello from systemd"
        sleep 10
done
```

**핵심 개념:** `systemd`는 서비스 관리자, `systemctl`은 제어 명령어다. `enabled`는 부팅 자동 시작 설정, `active (running)`은 현재 실행 상태다. `--no-pager`는 페이저 없이 출력한다.

## 3. 정상 중지 및 수동 복구

```bash
sudo systemctl stop hello-systemd.service
systemctl status hello-systemd.service --no-pager
sudo systemctl start hello-systemd.service
systemctl status hello-systemd.service --no-pager
```

- 중지 확인: `Active: inactive (dead)`
- 종료 로그: `Deactivated successfully`, `Stopped hello-systemd.service`
- 중지 시 프로세스: PID 1006, `signal=TERM`
- 수동 재시작 후: `Active: active (running)`, 새 PID 1513
- `hello from systemd` 로그가 10초 간격으로 다시 출력됨

**판정:** 관리자에 의한 정상 중지와 수동 복구 성공. 이 단계는 비정상 장애가 아니라 의도적 관리 작업이다.

## 4. 비정상 종료 주입 및 자동 복구 (장애 대응 사례 #1)

### 장애 재현

```bash
sudo systemctl kill --kill-whom=main --signal=SIGKILL hello-systemd.service
systemctl status hello-systemd.service --no-pager
journalctl -u hello-systemd.service -n 30 --no-pager
```

- **장애 주입:** 실습 서비스의 주 프로세스 PID 1513에 SIGKILL 전송
- **발생 시각:** 2026-11-12 05:14:53 UTC
- **영향 범위:** `hello-systemd.service` 실습 프로세스
- **예상 동작:** `Restart=on-failure`에 따라 systemd가 비정상 종료를 감지하고 서비스를 재시작

### 실제 검증 로그

```text
Nov 12 05:14:53 ubuntu-server systemd[1]: hello-systemd.service: Sent signal SIGKILL to main process 1513 (hello-systemd.s) on client request.
Nov 12 05:14:53 ubuntu-server systemd[1]: hello-systemd.service: Main process exited, code=killed, status=9/KILL
Nov 12 05:14:53 ubuntu-server systemd[1]: hello-systemd.service: Failed with result 'signal'.
Nov 12 05:14:53 ubuntu-server systemd[1]: hello-systemd.service: Scheduled restart job, restart counter is at 1.
Nov 12 05:14:53 ubuntu-server systemd[1]: Started hello-systemd.service - Hello Systemd Practice Service.
Nov 12 05:14:53 ubuntu-server hello-systemd.sh[1806]: hello from systemd
Nov 12 05:15:03 ubuntu-server hello-systemd.sh[1806]: hello from systemd
```

### 원인 → 조치 → 결과

1. **원인(실습 주입):** 관리자 요청에 따라 서비스 주 프로세스를 `SIGKILL`로 종료.
2. **탐지:** systemd가 `status=9/KILL`, `Failed with result 'signal'`로 실패를 감지.
3. **대응:** 설정된 `Restart=on-failure` 정책에 따라 자동 재시작.
4. **검증:** PID가 **1513 → 1806**으로 변경되고 반복 로그 출력 재개.
5. **결론:** 자동 복구 성공. 관련 로그는 같은 초에 기록되었으므로 정확한 밀리초 단위 복구 시간은 측정하지 못함.

> 참고: 이 사례는 의도적으로 주입한 장애다. 실제 운영 서비스 장애를 해결했다고 주장하지 않는다.

## 5. 오늘 사용한 핵심 명령어

```bash
hostnamectl
uptime
nproc
top
systemctl list-units --type=service --state=running --no-pager
systemctl status hello-systemd.service --no-pager
systemctl cat hello-systemd.service
cat /usr/local/bin/hello-systemd.sh
sudo systemctl stop hello-systemd.service
sudo systemctl start hello-systemd.service
sudo systemctl kill --kill-whom=main --signal=SIGKILL hello-systemd.service
journalctl -u hello-systemd.service -n 30 --no-pager
```

## 6. 학습 평가 및 다음 단계

- [x] SSH를 통해 기존 Ubuntu Server 접속
- [x] CPU·메모리·시스템 부하 확인
- [x] systemd 서비스 및 Unit 설정 분석
- [x] 정상 중지와 수동 복구 검증
- [x] SIGKILL 장애 재현과 자동 재시작 로그 검증
- [ ] Day 1 이해도 평가 3문항 제출 및 채점 (**미완료**)
- [ ] Day 2: `journalctl` 필터링, 오류 원인 분석 및 운영 장애 대응 심화

**학습 포인트:** 명령을 실행했다는 사실과 결과가 정상인지 검증했다는 사실을 구분한다. 서비스가 `active`여도 실제 기능 점검과 로그 확인이 필요하다.
