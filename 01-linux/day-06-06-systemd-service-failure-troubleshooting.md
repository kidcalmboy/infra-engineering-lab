# Day 6-6 — systemd Service 장애 분석과 Troubleshooting

> 연습용 `hello-systemd.service`에 의도적으로 장애를 만들어 **증상 → 가설 → 확인 → 원인 → 조치 → 검증 → 재발 방지** 흐름으로 systemd Service 장애를 분석하는 방법을 학습했다.

## 📌 이번에 배운 내용

- `Active: failed` 상태를 장애의 출발점으로 해석하는 방법
- `systemctl status`와 `journalctl -u`를 함께 보는 이유
- 잘못된 `ExecStart=` 경로 장애
- 실행 권한이 없는 Script의 `Permission denied` 장애
- Unit 파일 수정 후 `daemon-reload` 누락 문제
- `Restart=on-failure` 동작과 Process 재생성
- `systemctl reset-failed`의 의미와 한계
- Unit 변경과 Script 변경에서 `daemon-reload` 필요 여부 구분
- Service 장애를 무작정 restart/reboot하지 않고 원인을 먼저 찾는 운영 절차

## 📚 목차

1. Troubleshooting 기본 흐름
2. 실습 환경
3. 장애 1 — `ExecStart` 경로 오류
4. 장애 2 — Script 실행 권한 오류
5. 장애 3 — `daemon-reload` 누락
6. 장애 4 — `Restart=on-failure` 확인
7. `reset-failed`
8. `daemon-reload` 필요 여부 비교
9. 운영형 장애 대응 순서
10. 헷갈리기 쉬운 부분
11. 실무 포인트
12. 핵심 정리
13. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 목적 |
|---|---|
| `systemctl status hello-systemd` | 현재 Service 상태와 실패 단서 확인 |
| `journalctl -u hello-systemd -n 30 --no-pager` | 최근 Unit 로그 확인 |
| `systemctl cat hello-systemd` | 실제 Unit/Override와 `ExecStart` 확인 |
| `systemctl show hello-systemd -p MainPID` | Main PID 확인 |
| `ls -l /usr/local/bin/hello-system.sh` | Script 존재 여부와 실행 권한 확인 |
| `sudo systemctl daemon-reload` | Unit 정의 다시 읽기 |
| `sudo systemctl reset-failed hello-systemd` | failed 상태/실패 카운터 초기화 |
| `sudo kill -9 PID` | 비정상 종료 상황을 의도적으로 재현 |

---

# 1. Troubleshooting 기본 흐름

Service 장애 대응은 명령어를 무작정 반복하는 것이 아니라 다음 순서로 접근한다.

```text
증상
→ 가설
→ 확인
→ 원인
→ 조치
→ 검증
→ 재발 방지
```

운영에서는 다음처럼 구체화할 수 있다.

```text
systemctl status UNIT
→ journalctl -u UNIT
→ systemctl cat UNIT
→ systemctl show UNIT
→ 실제 파일 / 권한 / Process 확인
→ 원인 확정
→ 최소 조치
→ 필요하면 daemon-reload
→ start/restart
→ status/journal 재검증
→ 실제 기능 검증
```

---

# 2. 실습 환경

연습용 Service:

```text
hello-systemd.service
```

Unit 파일:

```text
/etc/systemd/system/hello-systemd.service
```

실제 Script:

```text
/usr/local/bin/hello-system.sh
```

정상적인 핵심 설정:

```ini
[Service]
Type=simple
ExecStart=/usr/local/bin/hello-system.sh
Restart=on-failure
```

> ⚠️ Service 이름과 Script 이름은 같을 필요가 없다. 중요한 것은 `ExecStart=`의 경로가 실제 실행 파일과 정확히 일치하는 것이다.

---

# 3. 장애 1 — `ExecStart` 경로 오류

## 증상 만들기

`ExecStart=`를 존재하지 않는 경로로 변경한다.

```ini
ExecStart=/usr/local/bin/hello-system-wrong.sh
```

Unit 파일을 수정했으므로:

```bash
sudo systemctl daemon-reload
sudo systemctl restart hello-systemd
```

또는 Service가 정지 상태라면:

```bash
sudo systemctl start hello-systemd
```

## 확인

```bash
systemctl status hello-systemd
journalctl -u hello-systemd -n 30 --no-pager
```

환경에 따라 다음과 비슷한 단서를 볼 수 있다.

```text
Active: failed
No such file or directory
Failed at step EXEC
```

문구는 systemd 버전에 따라 달라질 수 있으므로 **문자열 자체보다 의미를 해석하는 것**이 중요하다.

## 가설과 원인 확정

```bash
systemctl cat hello-systemd
ls -l /usr/local/bin/hello-system.sh
ls -l /usr/local/bin/hello-system-wrong.sh
```

판단:

```text
Unit의 ExecStart 경로
≠
실제 Script 경로
```

## 조치

```ini
ExecStart=/usr/local/bin/hello-system.sh
```

수정 후:

```bash
sudo systemctl daemon-reload
sudo systemctl restart hello-systemd
```

검증:

```bash
systemctl is-active hello-systemd
pgrep -af hello-system
journalctl -u hello-systemd -n 20 --no-pager
```

---

# 4. 장애 2 — Script 실행 권한 오류

## 증상 만들기

먼저 Service를 중지한다.

```bash
sudo systemctl stop hello-systemd
```

실행 권한 제거:

```bash
sudo chmod -x /usr/local/bin/hello-system.sh
ls -l /usr/local/bin/hello-system.sh
```

그 다음 시작:

```bash
sudo systemctl start hello-systemd
```

## 확인

```bash
systemctl status hello-systemd
journalctl -u hello-systemd -n 30 --no-pager
```

권한 문제라면 환경에 따라 다음과 비슷한 단서가 나타날 수 있다.

```text
Permission denied
Failed at step EXEC
```

## 원인

```text
ExecStart 경로는 맞음
→ 파일도 존재함
→ 하지만 execute(x) 권한 없음
```

## 조치

```bash
sudo chmod +x /usr/local/bin/hello-system.sh
sudo systemctl start hello-systemd
```

검증:

```bash
systemctl status hello-systemd
journalctl -u hello-systemd -n 20 --no-pager
```

### 중요한 점

이번에는 Unit 파일 자체를 수정하지 않았으므로:

```text
daemon-reload 불필요
```

이다.

---

# 5. 장애 3 — `daemon-reload` 누락

Unit 파일의 `Description=` 등을 수정하고 일부러 `daemon-reload`를 하지 않는다.

예:

```ini
Description=Changed Hello Systemd Practice Service
```

확인:

```bash
systemctl status hello-systemd
systemctl show hello-systemd -p Description
```

systemd는 이전에 읽은 Unit 정의를 계속 인식하고 있을 수 있고, 환경에 따라 Unit 파일이 disk에서 변경되었다는 경고가 보일 수도 있다.

이 문제의 본질:

```text
Disk의 Unit 정의
≠
systemd Manager가 현재 인식 중인 Unit 정의
```

조치:

```bash
sudo systemctl daemon-reload
systemctl show hello-systemd -p Description
```

## 핵심 구분

```text
daemon-reload
→ systemd가 Unit 정의를 다시 읽음

restart
→ 실제 Service Process를 다시 실행
```

---

# 6. 장애 4 — `Restart=on-failure` 확인

정상 상태에서 Service를 시작한다.

```bash
sudo systemctl start hello-systemd
```

Main PID 확인:

```bash
systemctl show hello-systemd -p MainPID
```

예:

```text
MainPID=1234
```

의도적으로 강제 종료:

```bash
sudo kill -9 1234
```

다시 확인:

```bash
systemctl status hello-systemd
systemctl show hello-systemd -p MainPID
```

`Restart=on-failure` 정책 때문에 Service가 새 Process를 생성해 다시 실행될 수 있다.

```text
기존 Main PID
→ SIGKILL
→ 비정상 종료
→ systemd가 실패 감지
→ Restart=on-failure
→ 새 Process 생성
→ Main PID 변경
```

반면:

```bash
sudo systemctl stop hello-systemd
```

는 systemd에 의도적인 Service 중지를 요청하는 것이므로 단순 Process 강제 종료와 의미가 다르다.

---

# 7. `systemctl reset-failed`

```bash
sudo systemctl reset-failed hello-systemd
```

## 한 줄 정의

**systemd가 기억 중인 Unit의 failed 상태 및 실패 관련 카운터를 초기화하는 명령이다.**

하지만:

```text
reset-failed
≠
장애 원인 해결
```

이다.

잘못된 `ExecStart`, Permission 문제 등이 그대로라면 다시 시작했을 때 다시 실패한다.

---

# 8. `daemon-reload` 필요 여부 비교

| 변경 내용 | `daemon-reload` |
|---|---:|
| `.service` Unit 파일 수정 | 필요 |
| Drop-in Override 수정 | 필요 |
| Script 내용만 수정 | 보통 불필요 |
| Script 실행 권한 변경 | 불필요 |
| Service Process만 비정상 종료 | 불필요 |

판단 기준은:

> **systemd가 읽는 Unit 정의 자체가 바뀌었는가?**

이다.

---

# 9. 운영형 장애 대응 순서

```text
1. systemctl status UNIT
   → 현재 증상 / Active 상태 / 실패 단서

2. journalctl -u UNIT
   → 실제 실패 메시지와 시간 순서

3. systemctl cat UNIT
   → ExecStart / Override / 설정 출처

4. systemctl show UNIT
   → systemd가 실제 인식한 Property

5. ls -l / ps / pgrep 등
   → 파일 존재/권한/Process 상태 확인

6. 가설과 증거 일치 여부 확인

7. 원인에 맞는 최소 조치

8. Unit 파일을 수정했다면 daemon-reload

9. start / restart

10. status + journal 재확인

11. 실제 기능 검증
```

---

# 10. ⚠️ 헷갈리기 쉬운 부분

## `failed`는 원인이 아니다

```text
Active: failed
```

는 결과 상태다. 실제 원인은 journal, Unit 설정, 파일, 권한, Process 등을 더 확인해야 한다.

## status만 보고 끝내면 안 된다

`systemctl status`는 빠른 요약이고 최근 로그 일부만 보여준다. 장애 원인 분석에는 `journalctl -u`를 함께 사용한다.

## restart부터 누르면 원인 정보가 흐려질 수 있다

운영에서는 먼저 상태와 로그를 확보한 뒤 조치하는 습관이 중요하다.

## `kill -9`과 `systemctl stop`은 의미가 다르다

```text
kill -9 PID
→ Process 수준의 강제 종료

systemctl stop UNIT
→ Service lifecycle 수준의 의도적 중지
```

---

# 11. 💼 실무 포인트

좋지 않은 흐름:

```text
서비스 이상
→ restart
→ 또 안 됨
→ reboot
```

권장 흐름:

```text
서비스 이상
→ 증상 보존
→ status
→ journal
→ 설정/파일/권한/Process 확인
→ 원인 특정
→ 최소 변경
→ 재검증
```

운영에서는 “서비스를 다시 살렸다”뿐 아니라 다음을 설명할 수 있어야 한다.

```text
무엇이 실패했는가
왜 실패했는가
어떤 근거로 원인을 판단했는가
무엇을 변경했는가
정상 복구를 어떻게 검증했는가
재발을 어떻게 막을 것인가
```

---

# 12. ✅ 핵심 정리

```text
ExecStart 경로 오류
→ No such file / EXEC 관련 단서
→ Unit과 실제 파일 경로 비교

실행 권한 오류
→ Permission denied
→ ls -l로 x 권한 확인

Unit 수정 후 반영 안 됨
→ daemon-reload 여부 확인

Process 비정상 종료
→ Restart=on-failure 정책 확인

failed 상태
→ 원인이 아니라 결과
```

Day 6 전체에서 가장 중요한 운영 흐름:

```text
status
→ journal
→ cat/show
→ file/process/permission 확인
→ 원인 특정
→ 최소 조치
→ daemon-reload 필요 여부 판단
→ start/restart
→ status/journal
→ 실제 기능 검증
```

---

# 13. 🧠 복습 문제

1. `Active: failed`가 장애의 원인을 의미하지 않는 이유는?
2. `ExecStart=`에 존재하지 않는 파일을 지정하면 무엇부터 확인해야 하는가?
3. 파일이 존재하지만 실행 권한이 없을 때 어떤 명령으로 확인할 수 있는가?
4. Script 권한만 변경했을 때 `daemon-reload`가 필요하지 않은 이유는?
5. Unit 파일을 수정하고 `daemon-reload`를 하지 않으면 어떤 문제가 생길 수 있는가?
6. `daemon-reload`와 `restart`의 차이는?
7. `reset-failed`가 실제 장애 원인을 해결하지 못하는 이유는?
8. `Restart=on-failure`가 설정된 Service의 Main PID를 `kill -9` 하면 어떤 흐름이 발생할 수 있는가?
9. `kill -9 PID`와 `systemctl stop UNIT`의 의미 차이는?
10. Service 장애를 발견했을 때 운영형 Troubleshooting 순서를 설명해보자.

---

## Day 6 완료

Day 6에서는 다음 흐름을 모두 학습했다.

```text
Service / Daemon / systemd
→ systemctl
→ status / active / enabled
→ journalctl
→ Unit 파일
→ Dependency / Ordering
→ ExecStart / Restart
→ enable / target / symlink
→ Override / cat / show
→ 직접 Service 생성
→ lifecycle 운영
→ failed 상태와 장애 분석
```

다음은 **Day 7 — Linux 패키지 관리: apt, dpkg, Repository**로 넘어간다.
