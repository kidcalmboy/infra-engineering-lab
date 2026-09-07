# Day 2-3 — `sed`와 설정 파일 수정

> `sed`를 이용해 텍스트를 치환하고, 설정 파일을 변경할 때 **현재 상태 확인 → 백업 → 미리보기 → 적용 → 차이 확인 → 서비스 검증** 순서로 작업하는 운영 습관을 학습했다.

## 📌 이번에 배운 내용

- `sed` = Stream Editor
- Pattern Space의 기초 개념
- `s/old/new/` 치환
- `g`, `p`, `d`
- `-n`, `-i`
- Address(줄 번호/범위/패턴) 지정
- `diff`로 변경점 확인
- 명령 종료 성공과 목표 상태 달성의 차이
- 설정 파일 변경 시 안전한 운영 절차

## 📚 목차

1. `sed`란 무엇인가
2. Stream Editor와 처리 흐름
3. 치환 문법
4. Address
5. 주요 옵션/명령
6. `-i`와 원본 수정
7. `diff`
8. 실제 실습
9. 헷갈리기 쉬운 부분
10. Troubleshooting
11. 실무 포인트
12. 핵심 정리
13. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 옵션/표현 | 의미 |
|---|---|---|
| `sed 's/dev/prod/' file` | `s` = substitute | 각 줄 첫 매칭 치환 결과 출력 |
| `sed 's/error/ERROR/g' file` | `g` = global | 한 줄의 모든 매칭 치환 |
| `sed -n '2p' file` | `-n` + `p` | 2번째 줄만 출력 |
| `sed -n '2,4p' file` | `2,4` = address range | 2~4번째 줄 출력 |
| `sed '/INFO/d' file` | `d` = delete | INFO 줄을 출력 결과에서 제거 |
| `sed -i 's/dev/prod/' file` | `-i` = in-place | 파일 자체 수정 |
| `diff before after` | 두 파일 인자 | 변경 전후 비교 |

---

## 1. `sed`란 무엇인가

`sed`는 **Stream Editor**다. 파일을 GUI Editor처럼 열어 커서를 이동하는 방식이 아니라, 입력되는 텍스트 stream에 규칙을 적용하여 결과를 출력한다.

기본 구조:

```text
입력
→ 한 줄 읽기
→ sed 명령 적용
→ 출력
→ 다음 줄
```

그래서 반복 가능한 자동 변환에 강하다.

---

## 2. Stream Editor와 Pattern Space

`sed`는 입력을 한 줄씩 읽어 내부 작업 공간인 **Pattern Space**에 넣고 명령을 적용한다. 기본적으로 명령 처리가 끝나면 Pattern Space 내용을 stdout으로 출력하고 다음 줄로 넘어간다.

초심자 관점에서는:

```text
파일 전체를 한 번에 직접 수정
```

하는 도구라기보다:

```text
각 줄을 읽음
→ 규칙 적용
→ 결과 출력
```

한다고 이해하면 된다.

`-i`를 붙일 때만 결과를 원본 파일에 반영하는 효과가 생긴다.

---

## 3. 치환 문법

기본:

```bash
sed 's/old/new/' file
```

` s `는 **substitute**다.

구조:

```text
s / 찾을패턴 / 바꿀문자열 /
```

예:

```bash
sed 's/mode=dev/mode=prod/' service.conf
```

각 줄에서 첫 번째 매칭만 바꾼 결과를 출력한다.

### `g` — global flag

```bash
sed 's/error/ERROR/g' file
```

한 줄 안에서 여러 번 매칭되면 전부 바꾼다.

```text
s/a/b/   → 한 줄의 첫 번째 a만
s/a/b/g  → 한 줄의 모든 a
```

`g`는 `sed` 실행 옵션이 아니라 **substitution command 뒤에 붙는 flag**라는 점도 구분한다.

---

## 4. Address

`sed`는 어떤 줄에 명령을 적용할지 Address를 지정할 수 있다.

### 줄 번호

```bash
sed -n '2p' file
```

2번째 줄.

### 범위

```bash
sed -n '2,4p' file
```

2~4번째 줄.

### 패턴

```bash
sed '/ERROR/p' file
```

`ERROR`가 있는 줄에 `p` 명령을 적용한다.

단, 기본 자동 출력이 켜져 있으므로 `p`만 쓰면 매칭 줄이 두 번 보일 수 있다. 필요한 줄만 보고 싶다면 보통:

```bash
sed -n '/ERROR/p' file
```

처럼 쓴다.

---

## 5. 주요 옵션과 명령

### `-n` — 자동 출력 억제

`sed`는 기본적으로 처리 후 각 줄을 자동 출력한다.

```bash
sed -n '2p' file
```

`-n`은 이 자동 출력을 끈다.

### `p` — print

명시적으로 Pattern Space를 출력한다.

### `d` — delete

현재 Pattern Space를 삭제하고 다음 입력으로 넘어간다.

```bash
sed '/INFO/d' file
```

`-i`가 없으면 원본 파일의 줄을 실제로 지우는 것이 아니라 **출력 결과에서 제외**한다.

### `-i` — in-place

```bash
sed -i 's/dev/prod/' file
```

파일 자체에 결과를 반영한다.

운영에서 가장 주의해야 하는 옵션이다.

---

## 6. `-i`와 원본 수정

### Preview

```bash
sed 's/mode=dev/mode=prod/' service.conf
```

원본은 그대로다.

### Apply

```bash
sed -i 's/mode=dev/mode=prod/' service.conf
```

원본을 수정한다.

### 안전한 흐름

```text
cat/grep로 현재 값 확인
→ cp로 백업
→ sed without -i로 preview
→ 결과 확인
→ sed -i로 적용
→ grep/cat로 확인
→ diff로 변경점 확인
```

운영에서는 파일 자체 변경이 끝이 아니다. 서비스 설정이라면 이후 문법 검사와 reload/restart, 상태/로그 검증까지 이어진다.

---

## 7. `diff`

### 한 줄 정의

`diff`는 두 텍스트 파일의 차이를 비교한다.

```bash
diff service.conf.bak service.conf
```

전통적 출력에서:

```text
a = add
c = change
d = delete
```

예:

```text
3c3
< mode=dev
---
> mode=prod
```

```text
< → 첫 번째 파일
> → 두 번째 파일
c → 변경
```

운영에서 중요한 건 “명령을 실행했다”가 아니라 **무엇이 실제로 바뀌었는지 확인**하는 것이다.

자주 보게 될 옵션:

- `-u`: unified diff 형식. Git diff와 비슷해 읽기 좋다.

```bash
diff -u before.conf after.conf
```

---

## 8. 🧪 실제 실습

초기 파일:

```text
server_name=web01
port=8080
mode=dev
log_level=INFO
```

확인:

```bash
cat service.conf
grep '^mode=' service.conf
```

백업:

```bash
cp service.conf service.conf.bak
```

Preview:

```bash
sed 's/mode=dev/mode=prod/' service.conf
```

적용:

```bash
sed -i 's/mode=dev/mode=prod/' service.conf
```

검증:

```bash
grep '^mode=' service.conf
diff -u service.conf.bak service.conf
```

---

## 9. ⚠️ 헷갈리기 쉬운 부분

> ⚠️ `sed`가 exit code 0으로 끝나도 원하는 문자열이 실제로 치환됐다는 뜻은 아니다. 검색 Pattern이 없으면 아무 변경 없이 정상 종료할 수 있다.

> ⚠️ `g`는 전체 파일(global file)을 뜻하는 게 아니라 **각 줄에서 모든 match를 치환**하는 flag다.

> ⚠️ `d`는 `-i` 없이 사용하면 원본 삭제가 아니라 출력 stream에서 제외한다.

> ⚠️ `-n`과 `p`는 함께 자주 쓴다. `-n`은 기본 출력 억제, `p`는 필요한 줄 명시 출력이다.

---

## 10. 🔧 Troubleshooting

### 증상

```bash
sed -i 's/mode=dev/mode=prod/' service.conf
```

를 실행했는데 내용이 바뀌지 않았다.

### 가설

- 이미 `mode=prod`였나?
- 공백/문자열 형식이 다른가?
- 대상 파일이 맞나?

### 확인

```bash
pwd
ls -l service.conf
grep '^mode=' service.conf
```

### 원인

치환 대상 `mode=dev`가 존재하지 않았다.

### 조치/검증

현재 값을 기준으로 필요한 변경을 다시 설계하고, Preview → Apply → `grep`/`diff`로 확인한다.

### 배운 점

```text
명령 실행 성공
≠
의도한 상태 달성
```

운영자는 항상 **목표 상태 검증**까지 해야 한다.

---

## 11. 💼 실무 포인트

설정 변경의 일반적인 운영 흐름:

```text
현재 상태 확인
→ 변경 필요성 판단
→ 백업
→ Preview
→ 파일 수정
→ Syntax Test
→ reload/restart
→ service status
→ log 확인
→ 기능 검증
```

예를 들어 Nginx라면 설정 수정 후 `nginx -t` 같은 별도 문법 검증이 필요할 수 있다. `sed -i` 성공만 보고 변경 완료라고 판단하면 안 된다.

---

## 12. ✅ 핵심 정리

- `sed`는 Stream Editor다.
- `s/old/new/`는 치환 명령이고 `g`는 한 줄의 모든 매칭을 대상으로 한다.
- `-n`은 자동 출력을 끄고 `p`는 명시적으로 출력한다.
- `d`는 Pattern Space를 버린다.
- `-i`는 원본을 직접 변경한다.
- `diff`는 변경 전후 차이를 검증한다.
- 명령 성공과 목표 상태 달성은 다르다.

---

## 13. 🧠 복습 문제

1. Stream Editor란 무엇인가?
2. Pattern Space는 어떤 역할을 하는가?
3. `s/a/b/`와 `s/a/b/g`의 차이는?
4. `sed -n '2,4p'`를 해석해보라.
5. `d`가 `-i` 없이 사용될 때 원본 파일은 어떻게 되는가?
6. `diff -u`는 언제 유용한가?
7. 왜 설정 파일 변경 후 서비스 검증까지 해야 하는가?
