# Day 2 - sed와 설정 파일 수정 실습

## 학습 목표

- `sed`의 기본 치환 문법을 이해한다.
- 원본을 바꾸지 않고 치환 결과만 확인하는 방법과 `-i`로 실제 파일을 수정하는 방법을 구분한다.
- `g`, `-n`, `p`, `d` 옵션을 사용해 텍스트를 치환·출력·삭제한다.
- 설정 파일 수정 전 백업하고, 수정 후 결과를 검증하는 운영 습관을 익힌다.

## `sed` 기본 구조

`sed`는 stream editor로, 텍스트를 명령어로 변환할 때 사용한다.

기본 치환 문법:

```bash
sed 's/기존값/새값/' 파일
```

예:

```bash
sed 's/dev/prod/' service.conf
```

이 명령은 화면에 치환된 결과만 출력하고 원본 파일은 수정하지 않는다.

실제로 파일을 수정하려면 `-i` 옵션을 사용한다.

```bash
sed -i 's/dev/prod/' service.conf
```

## 전체 일치값 치환

기본 `s///`는 각 줄에서 첫 번째 일치만 변경한다.

```bash
sed 's/error/ERROR/' test.txt
```

한 줄에 있는 모든 일치값을 바꾸려면 `g`를 붙인다.

```bash
sed 's/error/ERROR/g' test.txt
```

- `s`: substitute
- `g`: global, 한 줄의 모든 일치값 치환

## 특정 줄 출력

두 번째 줄만 출력:

```bash
sed -n '2p' service.conf
```

2~4번째 줄 출력:

```bash
sed -n '2,4p' service.conf
```

- `-n`: 기본 출력을 억제
- `p`: 지정한 줄 출력

## 특정 줄 삭제

화면 출력에서 2번째 줄을 제외:

```bash
sed '2d' service.conf
```

`INFO`가 포함된 줄을 제외:

```bash
sed '/INFO/d' service.conf
```

`-i`가 없으면 원본 파일은 그대로이고 결과만 화면에 출력된다.

## `grep`, `awk`, `sed` 역할 구분

```text
grep → 문자열이 포함된 줄 검색
awk  → 필드 기반 조건 처리와 출력/계산
sed  → 문자열 치환, 삭제, 간단한 텍스트 변환
```

예:

```bash
grep "ERROR" server.log
awk '$4 >= 400 {print $1}' access.log
sed 's/8080/80/' service.conf
```

## 설정 파일 수정 실습

실습 파일:

```text
server_name=web01
port=8080
mode=prod
log_level=INFO
```

운영 환경 설정을 변경하는 상황을 가정하고 다음 순서로 접근했다.

### 1. 현재 내용 확인

```bash
cat service.conf
```

### 2. 백업 생성

```bash
cp service.conf service.conf.bak
```

### 3. 설정 수정

예를 들어 `mode=dev`를 `mode=prod`로 바꾸는 경우:

```bash
sed -i 's/mode=dev/mode=prod/' service.conf
```

### 4. 수정 결과 확인

```bash
cat service.conf
```

## Troubleshooting에서 확인한 점

실습 화면에서는 `sed` 실행 전부터 이미 다음 상태였다.

```text
mode=prod
```

이 상태에서 아래 명령을 실행하면:

```bash
sed -i 's/mode=dev/mode=prod/' service.conf
```

치환 대상인 `mode=dev`가 없기 때문에 파일 내용은 변경되지 않는다. `sed`는 일치하는 문자열이 없더라도 일반적으로 오류를 출력하지 않고 종료한다.

따라서 수정 명령을 실행했다는 사실만으로 변경이 성공했다고 판단하면 안 되고, 수정 전후 파일 내용을 직접 확인해야 한다.

실제 장애 상황을 재현하려면 먼저:

```bash
sed -i 's/mode=prod/mode=dev/' service.conf
```

로 잘못된 설정을 만든 뒤 다음 순서로 복구할 수 있다.

```bash
cat service.conf
cp service.conf service.conf.bak
sed -i 's/mode=dev/mode=prod/' service.conf
cat service.conf
```

## 변경점 검증

백업 파일과 현재 파일의 차이를 비교할 때는 `diff`를 사용할 수 있다.

```bash
diff service.conf.bak service.conf
```

예상 형태:

```text
< mode=dev
---
> mode=prod
```

`diff`는 이번 단계에서 보조 명령어로만 익히고, 핵심은 수정 전 백업과 수정 후 검증이다.

## 운영 관점에서 배운 점

설정 파일 수정은 다음 흐름으로 접근한다.

```text
현재 상태 확인
→ 원본 백업
→ 수정
→ 수정 결과 확인
→ 필요 시 diff로 변경점 검증
→ 서비스 반영 후 상태/로그 확인
```

특히 `sed -i`는 원본 파일을 즉시 수정하므로 운영 서버에서는 중요한 설정 파일에 바로 사용하기 전에 백업을 먼저 만드는 습관이 필요하다.

또한 치환 대상 문자열이 실제로 존재하는지 확인하지 않으면 명령은 정상 종료돼도 아무 변경도 일어나지 않을 수 있다.
