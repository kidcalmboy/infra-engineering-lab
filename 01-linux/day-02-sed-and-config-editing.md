# Day 2 - sed와 설정 파일 수정 실습

## 학습 목표

- `sed`가 무엇을 하는 명령어인지 단순 암기가 아니라 입력 → 처리 → 출력 흐름으로 이해한다.
- `sed 's/old/new/'`가 원본 파일을 직접 바꾸는 것이 아니라 변환 결과를 표준 출력으로 내보낸다는 점을 이해한다.
- `-i`가 왜 위험할 수 있는지, 운영 환경에서 왜 백업과 검증이 필요한지 이해한다.
- `s`, `g`, `-n`, `p`, `d`의 역할을 구분한다.
- `grep`, `awk`, `sed`를 각각 어떤 상황에서 선택해야 하는지 구분한다.
- 설정 파일 변경 시 확인 → 백업 → 미리보기 → 수정 → 비교 → 검증 순서로 작업하는 습관을 만든다.

## `sed`란 무엇인가

`sed`는 **Stream Editor**의 약자다. 텍스트를 한 줄씩 읽으면서 지정한 규칙에 따라 치환, 삭제, 선택 출력 등의 처리를 수행하는 명령어다.

핵심은 `sed`가 기본적으로 파일을 텍스트 편집기처럼 열어서 직접 수정하는 프로그램이 아니라는 점이다.

```text
입력 파일
   ↓
sed가 한 줄씩 읽음
   ↓
지정한 규칙 적용
   ↓
처리된 결과를 화면(표준 출력)에 출력
```

예를 들어:

```bash
sed 's/dev/prod/' service.conf
```

을 실행하면 `service.conf`를 읽어 `dev`를 `prod`로 바꾼 결과를 화면에 보여준다. 이 단계에서는 원본 파일 자체는 바뀌지 않는다.

따라서 `sed`를 처음 사용할 때는 **먼저 `-i` 없이 결과를 미리 확인하고, 원하는 결과가 맞을 때만 실제 파일을 수정하는 방식**이 안전하다.

## 기본 치환 문법

```bash
sed 's/기존값/새값/' 파일
```

여기서:

```text
s
→ substitute, 문자열 치환 명령

기존값
→ 찾을 문자열 또는 패턴

새값
→ 바꿀 문자열

/
→ 각 요소를 나누는 구분자
```

예:

```bash
sed 's/dev/prod/' service.conf
```

입력:

```text
mode=dev
```

출력:

```text
mode=prod
```

하지만 원본을 다시 확인하면:

```bash
cat service.conf
```

여전히 `mode=dev`일 수 있다. `-i`를 사용하지 않았기 때문이다.

## 기본 `s///`는 각 줄의 첫 번째 일치만 치환한다

파일 내용이 다음과 같다고 하자.

```text
error error error
info error info
```

실행:

```bash
sed 's/error/ERROR/' test.txt
```

결과:

```text
ERROR error error
info ERROR info
```

`sed`는 파일 전체에서 딱 한 번만 바꾼 것이 아니라 **각 줄마다 첫 번째로 일치한 문자열 하나를 바꾼 것**이다.

한 줄 안의 모든 일치값을 바꾸려면 `g` 플래그를 붙인다.

```bash
sed 's/error/ERROR/g' test.txt
```

결과:

```text
ERROR ERROR ERROR
info ERROR info
```

```text
g = global
→ 한 줄에서 일치하는 모든 값을 치환
```

## `-i` - 원본 파일을 실제로 수정

```bash
sed -i 's/dev/prod/' service.conf
```

`-i`는 **in-place** 수정 옵션이다. 처리 결과를 화면에만 보여주는 것이 아니라 원본 파일에 반영한다.

```text
sed 's/dev/prod/' file
→ 결과만 출력
→ 원본 유지

sed -i 's/dev/prod/' file
→ 원본 파일 자체 수정
```

운영 서버에서는 `-i`가 특히 중요하고 위험하다. 설정 파일에서 잘못된 패턴을 치환하면 서비스가 기동되지 않거나 예상과 다른 설정이 적용될 수 있기 때문이다.

그래서 다음 습관을 기본으로 잡는다.

```text
1. 현재 내용 확인
2. 원본 백업
3. -i 없이 결과 미리보기
4. 원하는 결과인지 확인
5. -i로 실제 수정
6. 변경점 비교
7. 설정/서비스 검증
```

## `-n`과 `p` - 필요한 줄만 출력

`sed`는 기본적으로 입력받은 모든 줄을 처리 후 출력한다.

특정 줄만 보고 싶다면 기본 출력을 억제한 뒤 원하는 줄만 `p`로 출력할 수 있다.

```bash
sed -n '2p' service.conf
```

```text
-n
→ 기본 자동 출력을 하지 않음

2
→ 두 번째 줄 선택

p
→ 선택된 줄 print
```

2~4번째 줄:

```bash
sed -n '2,4p' service.conf
```

이 구조는 긴 설정 파일에서 특정 구간만 빠르게 확인할 때 사용할 수 있다.

## `d` - 출력 흐름에서 줄 제거

두 번째 줄을 제외하고 출력:

```bash
sed '2d' service.conf
```

`INFO`가 포함된 줄을 제외:

```bash
sed '/INFO/d' service.conf
```

여기서 중요한 점은 `-i`가 없다면 **원본 파일에서 실제로 삭제하는 것이 아니라 출력 결과에서만 제외하는 것**이라는 점이다.

```text
sed '/INFO/d' file
→ INFO 줄을 뺀 결과를 화면에 출력
→ 원본 유지

sed -i '/INFO/d' file
→ 원본에서 해당 줄을 실제로 제거
```

따라서 `d`도 `-i`와 함께 사용하면 파괴적인 변경이 될 수 있으므로 주의해야 한다.

## 주소(Address) 개념

`sed`에서는 어떤 줄에 명령을 적용할지 지정하는 부분을 주소라고 생각하면 이해하기 쉽다.

예:

```bash
sed -n '3p' file
```

→ 3번째 줄이라는 **줄 번호 조건**에 `p` 적용

```bash
sed '/ERROR/d' file
```

→ `ERROR`라는 **패턴 조건**에 맞는 줄에 `d` 적용

즉 `sed`는 단순 문자열 치환만 하는 명령어가 아니라:

```text
어떤 줄을 선택할지
+
그 줄에 어떤 동작을 할지
```

를 조합해 사용하는 도구다.

## `grep`, `awk`, `sed` 역할 구분

세 명령은 겹치는 기능이 있지만 주 목적이 다르다.

```text
grep
→ 원하는 문자열/패턴이 있는 줄을 찾는다.

awk
→ 한 줄을 필드로 나누고 조건 비교, 출력, 계산을 한다.

sed
→ 텍스트를 규칙에 따라 치환하거나 삭제하는 등 변환한다.
```

예:

```bash
grep "ERROR" server.log
```

→ ERROR 줄을 찾는 것이 목적

```bash
awk '$4 >= 400 {print $1}' access.log
```

→ 상태코드라는 필드 값을 비교하고 IP 필드를 출력하는 것이 목적

```bash
sed 's/8080/80/' service.conf
```

→ 설정 문자열을 다른 값으로 바꾸는 것이 목적

단순 검색인데 억지로 `sed`를 쓰거나, 단순 치환인데 복잡한 `awk`를 쓰기보다 **문제에 맞는 도구를 선택하는 것이 운영 명령어 사용의 핵심**이다.

## 설정 파일 수정 실습

실습 파일:

```text
server_name=web01
port=8080
mode=dev
log_level=INFO
```

### 1. 현재 설정 확인

```bash
cat service.conf
```

수정 전 현재 값이 무엇인지 확인한다. 운영 환경에서는 현재 상태를 모른 채 변경부터 하면 장애 원인 추적이 어려워진다.

### 2. 백업 생성

```bash
cp service.conf service.conf.bak
```

백업은 변경 실패 시 원래 상태로 복구할 기준점이 된다.

### 3. 실제 수정 전에 미리보기

```bash
sed 's/mode=dev/mode=prod/' service.conf
```

출력 결과가 의도와 일치하는지 확인한다.

### 4. 실제 수정

```bash
sed -i 's/mode=dev/mode=prod/' service.conf
```

### 5. 수정 결과 확인

```bash
cat service.conf
```

또는 필요한 항목만:

```bash
grep '^mode=' service.conf
```

## `sed`가 조용히 아무것도 바꾸지 않을 수 있다

실습에서는 이미 파일이 다음 상태였다.

```text
mode=prod
```

그런데 다음 명령을 실행했다.

```bash
sed -i 's/mode=dev/mode=prod/' service.conf
```

이 경우 찾을 문자열인 `mode=dev`가 없기 때문에 변경되는 내용이 없다.

중요한 점은 **치환 대상이 없다고 해서 `sed`가 반드시 오류를 출력하는 것은 아니라는 점**이다.

즉:

```text
명령어가 에러 없이 끝남
≠
원하는 변경이 실제로 일어남
```

따라서 운영에서는 명령 실행 성공 여부만 보지 말고 **실제 시스템 상태를 검증**해야 한다.

이 개념은 앞으로 서비스 운영 전체에 적용된다.

```text
명령 실행
→ 결과 확인
→ 실제 상태 확인
```

## `diff` - 변경 전후 비교

백업 파일과 수정된 파일의 차이를 비교한다.

```bash
diff service.conf.bak service.conf
```

예:

```text
3c3
< mode=dev
---
> mode=prod
```

여기서:

```text
3c3
→ 첫 번째 파일 3번째 줄이 두 번째 파일 3번째 줄로 변경됨

c = change
< = 첫 번째 파일 내용
> = 두 번째 파일 내용
```

`diff`에서 자주 볼 수 있는 문자:

```text
c = change, 변경
a = add, 추가
d = delete, 삭제
```

설정 파일을 수정할 때 `cat`으로 전체를 보는 것보다 `diff`를 사용하면 **실제로 무엇이 달라졌는지** 더 명확하게 확인할 수 있다.

## 왜 백업과 `diff`가 중요한가

운영 환경에서 설정 변경의 핵심은 단순히 값을 바꾸는 것이 아니다.

중요한 것은:

```text
변경 전 상태를 알고 있는가?
무엇을 바꿨는가?
의도한 값만 바뀌었는가?
문제가 생기면 원복할 수 있는가?
```

이다.

따라서 설정 작업은 다음처럼 접근한다.

```text
현재 설정
   ↓
백업본 생성
   ↓
변경
   ↓
diff로 변경 범위 확인
   ↓
설정 문법/서비스 상태 확인
   ↓
로그 확인
```

## Troubleshooting 실습

### 상황

서비스가 운영 환경인데 설정 파일에 다음 값이 들어 있다.

```text
mode=dev
```

정상값은:

```text
mode=prod
```

이다.

### 확인

```bash
cat service.conf
```

### 백업

```bash
cp service.conf service.conf.bak
```

### 수정 전 미리보기

```bash
sed 's/mode=dev/mode=prod/' service.conf
```

### 실제 수정

```bash
sed -i 's/mode=dev/mode=prod/' service.conf
```

### 검증

```bash
grep '^mode=' service.conf
diff service.conf.bak service.conf
```

이 실습의 핵심은 `sed` 문법 자체보다 **변경 작업을 안전하게 수행하는 절차**다.

## 실무에서 주의할 점

`sed -i`를 사용할 때 치환 범위가 너무 넓으면 의도하지 않은 곳까지 바뀔 수 있다.

예를 들어 단순히:

```bash
sed -i 's/80/443/g' service.conf
```

처럼 숫자만 치환하면 포트 이외의 다른 값에 포함된 `80`까지 바뀔 수 있다.

가능하면 문맥을 포함해 더 구체적으로 지정한다.

```bash
sed -i 's/port=8080/port=80/' service.conf
```

즉 운영 설정 변경에서는 **패턴을 가능한 한 구체적으로 작성하는 것**이 안전하다.

## 핵심 정리

```text
sed
→ 텍스트 스트림을 규칙에 따라 변환

s
→ substitute, 치환

g
→ 한 줄의 모든 일치값에 적용

-n
→ 자동 출력 억제

p
→ 선택한 줄 출력

d
→ 선택한 줄을 처리 결과에서 제거

-i
→ 원본 파일을 실제로 수정

diff
→ 변경 전후 차이 확인
```

운영 설정 변경의 기본 흐름:

```text
확인
→ 백업
→ 미리보기
→ 실제 수정
→ diff 비교
→ 설정/서비스 검증
→ 로그 확인
```
