# Day 2 - 파일 검색과 텍스트 처리 기초

## 학습 목표

- `find`로 파일과 디렉터리를 검색한다.
- `grep`으로 파일 내용에서 필요한 문자열만 찾는다.
- `wc -l`로 결과의 줄 수를 계산한다.
- `|` 파이프를 이용해 여러 명령어를 연결한다.
- 서버 로그에서 ERROR/WARNING 등 필요한 정보만 빠르게 추출하는 습관을 만든다.

## 핵심 개념

### `find`와 `grep`의 차이

- `find`: **파일 자체를 찾는 명령어**
- `grep`: **파일 내용에서 문자열을 찾는 명령어**

예를 들어 `.log` 파일의 위치를 찾고 싶다면 `find`, 특정 로그 파일 안에서 `ERROR`를 찾고 싶다면 `grep`을 사용한다.

## `find` 실습

홈 디렉터리 아래에서 특정 파일을 찾았다.

```bash
find ~ -name "server.log"
```

`.log` 확장자를 가진 파일 전체 검색:

```bash
find ~ -name "*.log"
```

대소문자를 구분하지 않고 검색할 때는 `-iname`을 사용할 수 있다.

```bash
find ~ -iname "SERVER.LOG"
```

## `grep` 실습

`server.log`에서 문자열을 기준으로 필요한 줄만 추출했다.

```bash
grep "ERROR" server.log
grep "WARNING" server.log
grep -i "error" server.log
grep -n "ERROR" server.log
grep -v "INFO" server.log
```

옵션의 의미:

- `-i`: 대소문자 구분 없이 검색
- `-n`: 일치한 줄의 줄 번호도 출력
- `-v`: 해당 문자열이 **없는 줄**을 출력

실습 문제에서는 다음 명령으로 `WARNING` 로그만 확인했다.

```bash
grep "WARNING" server.log
```

`INFO`가 아닌 로그만 확인:

```bash
grep -v "INFO" server.log
```

## 파이프 `|`

파이프는 왼쪽 명령어의 출력 결과를 오른쪽 명령어의 입력으로 전달한다.

```text
명령어 1 출력 → | → 명령어 2 입력
```

예:

```bash
history | grep "cd"
```

전체 명령어 기록 중 `cd`가 포함된 줄만 출력한다.

또한 다음과 같이 로그 필터링 결과를 다른 명령으로 다시 처리할 수 있다.

```bash
grep "ERROR" server.log | wc -l
```

## `wc -l` 실습

`wc`는 입력 데이터의 개수를 계산하는 명령어이고, `-l`은 줄 수만 출력한다.

```bash
wc -l server.log
```

`grep`과 결합하면 특정 로그가 몇 줄 발생했는지 계산할 수 있다.

```bash
grep "ERROR" server.log | wc -l
```

동작 흐름:

```text
server.log
→ grep으로 ERROR 줄만 추출
→ wc -l로 추출된 줄 개수 계산
```

## 직접 실습

기존 `server.log`에 로그를 추가한 뒤 검색과 집계를 진행했다.

```bash
cd ~/linux-test

echo "ERROR disk full" >> server.log
echo "INFO cleanup started" >> server.log
echo "ERROR backup failed" >> server.log
echo "WARNING memory usage 90%" >> server.log
```

검색:

```bash
grep "ERROR" server.log
grep -n "ERROR" server.log
grep -i "error" server.log
grep -v "INFO" server.log
```

집계:

```bash
grep "ERROR" server.log | wc -l
```

명령어 기록 검색:

```bash
history | grep "cd"
```

## Troubleshooting 실습

### 상황

애플리케이션 장애가 발생했고 로그 파일의 위치가 다음과 같다고 가정했다.

```text
/var/log/app/server.log
```

요구사항은 `ERROR` 로그가 존재하는지 확인하고 총 몇 건인지 파악하는 것이다.

먼저 실제 오류 내용을 확인한다.

```bash
grep "ERROR" /var/log/app/server.log
```

그 다음 건수를 계산한다.

```bash
grep "ERROR" /var/log/app/server.log | wc -l
```

단순히 개수만 확인하는 것보다 **오류 내용 확인 → 발생 건수 계산** 순서로 접근하는 것이 원인 분석에 더 유용하다.

## 배운 점

- 파일의 위치를 모르면 `find`, 파일 내용에서 값을 찾고 싶으면 `grep`을 사용한다.
- `grep` 옵션을 이용하면 대소문자 무시, 줄 번호 표시, 반대 조건 필터링이 가능하다.
- `wc -l`은 단독으로도 사용할 수 있지만 파이프와 결합할 때 특히 유용하다.
- `|`를 이용하면 한 명령어의 결과를 다음 명령어로 넘겨 원하는 정보만 단계적으로 가공할 수 있다.
- 로그 장애 분석에서는 결과 개수만 보는 것이 아니라 실제 ERROR 내용을 먼저 확인하는 습관이 중요하다.

앞으로 로그 분석 시 다음 흐름을 기본으로 사용한다.

```text
로그 위치 확인 → 필요한 문자열 검색 → 실제 내용 확인 → 필요 시 개수 집계
```
