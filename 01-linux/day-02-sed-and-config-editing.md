# Day 2-3 - `sed`와 설정 파일 수정

> `sed`를 이용해 텍스트를 치환하고, 설정 파일을 수정할 때 백업·미리보기·검증을 포함한 안전한 운영 절차를 학습했다.

## 📌 이번에 배운 내용

- `sed`의 Stream Editor 개념
- `s/기존값/새값/` 치환 문법
- `g`, `-n`, `p`, `d`, `-i`
- `sed -i`가 원본 파일을 직접 변경한다는 의미
- 변경 전 백업과 변경 후 `diff` 검증
- 치환 대상이 없어도 오류 없이 끝날 수 있다는 점
- 설정 파일 변경 시 운영 절차

## 📚 목차

1. `sed`란 무엇인가
2. 치환 문법
3. 주요 옵션과 명령
4. 원본 수정과 미리보기
5. `diff` 검증
6. 실습
7. 헷갈리기 쉬운 부분
8. Troubleshooting
9. 실무 포인트
10. 핵심 정리
11. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 용도 |
|---|---|
| `sed 's/dev/prod/' file` | 각 줄의 첫 번째 일치값을 치환하여 출력 |
| `sed 's/error/ERROR/g' file` | 각 줄의 모든 일치값 치환 |
| `sed -n '2p' file` | 2번째 줄만 출력 |
| `sed -n '2,4p' file` | 2~4번째 줄만 출력 |
| `sed '2d' file` | 출력 결과에서 2번째 줄 제거 |
| `sed -i 's/dev/prod/' file` | 원본 파일 직접 수정 |
| `diff old new` | 두 파일의 차이 확인 |

---

## 1. `sed`란 무엇인가

### 한 줄 정의

`sed`는 **Stream Editor**로, 입력되는 텍스트 흐름에 규칙을 적용해 출력하는 도구다.

### 상세 설명

기본 동작은 다음과 같이 이해할 수 있다.

```text
입력 파일
→ sed가 한 줄씩 처리
→ 지정한 규칙 적용
→ 결과를 표준 출력
```

`-i`를 사용하지 않으면 보통 원본 파일은 그대로이고 처리 결과만 화면에 출력된다.

### 실무에서 중요한 이유

설정 파일의 특정 값을 자동으로 바꾸거나 반복 작업을 스크립트로 처리할 때 유용하다. 다만 원본을 직접 수정하는 `-i`는 운영 환경에서 주의가 필요하다.

---

## 2. 치환 문법

기본 구조:

```bash
sed 's/기존값/새값/' 파일
```

`s`는 **substitute**의 의미다.

예:

```bash
sed 's/dev/prod/' service.conf
```

각 줄에서 첫 번째 `dev`를 `prod`로 바꾼 결과를 출력한다.

### 한 줄의 모든 일치값 치환

```bash
sed 's/error/ERROR/g' test.txt
```

`g`는 **global**로, 해당 줄에 존재하는 모든 일치값을 치환한다.

| 형태 | 의미 |
|---|---|
| `s/a/b/` | 각 줄의 첫 번째 `a`만 `b`로 변경 |
| `s/a/b/g` | 각 줄의 모든 `a`를 `b`로 변경 |

---

## 3. 주요 옵션과 명령

### `-n`과 `p`

기본 출력은 억제하고 필요한 줄만 출력한다.

```bash
sed -n '2p' service.conf
```

```bash
sed -n '2,4p' service.conf
```

- `-n` → 기본 출력 억제
- `p` → print

### `d`

지정한 줄을 출력 결과에서 삭제한다.

```bash
sed '2d' service.conf
```

`INFO`가 포함된 줄 제외:

```bash
sed '/INFO/d' service.conf
```

`-i`가 없다면 원본 파일이 삭제되는 것이 아니라 **출력 결과에서 제외**되는 것이다.

### `-i`

```bash
sed -i 's/dev/prod/' service.conf
```

`-i`는 in-place 편집으로, 처리 결과를 원본 파일에 반영한다.

> `sed -i`는 되돌리기 어려운 변경을 만들 수 있으므로 중요한 설정 파일에서는 먼저 백업하는 습관이 필요하다.

---

## 4. 원본 수정과 미리보기

### 미리보기

```bash
sed 's/mode=dev/mode=prod/' service.conf
```

원본은 바뀌지 않는다.

### 실제 수정

```bash
sed -i 's/mode=dev/mode=prod/' service.conf
```

원본을 직접 수정한다.

### 안전한 변경 흐름

```bash
cat service.conf
cp service.conf service.conf.bak
sed 's/mode=dev/mode=prod/' service.conf
sed -i 's/mode=dev/mode=prod/' service.conf
cat service.conf
```

즉:

```text
현재 상태 확인
→ 백업
→ 변경 결과 미리보기
→ 실제 수정
→ 결과 확인
```

---

## 5. `diff`로 변경점 검증

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

### `3c3` 의미

```text
3 c 3
│ │ │
│ │ └─ 두 번째 파일의 3번째 줄
│ └─── change
└───── 첫 번째 파일의 3번째 줄
```

- `<` → 첫 번째 파일 내용
- `>` → 두 번째 파일 내용
- `c` → change
- `a` → add
- `d` → delete

`diff`를 사용하면 “명령을 실행했다”가 아니라 **실제로 어떤 값이 바뀌었는지** 확인할 수 있다.

---

## 6. 🧪 실습

실습 파일:

```text
server_name=web01
port=8080
mode=dev
log_level=INFO
```

### 1. 현재 상태 확인

```bash
cat service.conf
```

### 2. 백업

```bash
cp service.conf service.conf.bak
```

### 3. 변경 미리보기

```bash
sed 's/mode=dev/mode=prod/' service.conf
```

### 4. 실제 변경

```bash
sed -i 's/mode=dev/mode=prod/' service.conf
```

### 5. 결과 검증

```bash
cat service.conf
diff service.conf.bak service.conf
```

### 결과

`mode=dev`가 `mode=prod`로 변경됐고, `diff`로 실제 변경 줄을 확인했다.

---

## 7. ⚠️ 헷갈리기 쉬운 부분

### `sed` 명령이 에러 없이 끝났으면 치환이 성공한 것인가?

아니다.

예를 들어 파일이 이미:

```text
mode=prod
```

상태인데:

```bash
sed -i 's/mode=dev/mode=prod/' service.conf
```

를 실행하면 일치하는 `mode=dev`가 없으므로 아무 변경도 일어나지 않을 수 있다. 그래도 일반적으로 명령 자체가 오류로 종료되는 것은 아니다.

> **명령 성공 여부와 원하는 설정 변경 성공 여부는 다르다.**

### `sed '/INFO/d'`는 파일에서 실제 줄을 지우는가?

`-i`가 없다면 아니다. 화면 출력에서만 제외한다.

### `grep`, `awk`, `sed`는 어떻게 구분하는가?

| 명령 | 적합한 작업 |
|---|---|
| `grep` | 조건에 맞는 줄 찾기 |
| `awk` | 필드 기반 조건·계산 |
| `sed` | 치환·삭제·텍스트 변환 |

---

## 8. 🔧 Troubleshooting

### 문제

설정을 `mode=dev`에서 `mode=prod`로 바꾸려고 했는데 `sed` 실행 전부터 이미 `mode=prod`였다.

### 증상

명령은 정상 종료됐지만 파일 내용이 변하지 않았다.

### 원인

치환 대상 문자열 `mode=dev`가 파일에 존재하지 않았다.

### 확인 방법

```bash
cat service.conf
grep '^mode=' service.conf
```

### 해결

현재 상태를 먼저 확인하고 실제로 필요한 변경인지 판단한다.

변경 작업에서는 다음 절차를 적용한다.

```text
현재 값 확인
→ 백업
→ 치환 대상 확인
→ 미리보기
→ 수정
→ diff 검증
```

### 배운 점

운영에서 중요한 것은 “명령을 실행했다”가 아니라 **원하는 상태가 실제로 만들어졌는지 검증하는 것**이다.

---

## 9. 💼 실무 포인트

실제 서비스 설정 변경은 파일 수정에서 끝나지 않는다.

일반적인 흐름:

```text
현재 설정 확인
→ 백업
→ 수정
→ 설정 파일 문법 검증
→ 서비스에 반영
→ 서비스 상태 확인
→ 로그 확인
```

예를 들어 Nginx 같은 서비스는 수정 후 별도의 문법 검증과 reload/restart 과정이 필요할 수 있다.

따라서 `sed -i`는 단순 편집 도구이고, **서비스가 정상 동작하는지까지 확인해야 변경 작업이 완료된 것**이다.

---

## 10. ✅ 핵심 정리

- `sed`는 Stream Editor이며 텍스트 입력에 규칙을 적용한다.
- `s/old/new/`는 각 줄의 첫 번째 일치값을 치환한다.
- `g`를 붙이면 각 줄의 모든 일치값을 치환한다.
- `-n`은 기본 출력을 억제하고 `p`는 지정한 내용을 출력한다.
- `d`는 처리 결과에서 줄을 제외한다.
- `-i`는 원본 파일을 직접 수정한다.
- 중요한 설정 파일 수정 전에는 백업하고, 수정 후 `cat`, `grep`, `diff` 등으로 검증한다.
- 명령이 성공적으로 종료됐다고 해서 원하는 설정 변경이 실제로 발생한 것은 아니다.

---

## 11. 🧠 복습 문제

1. `sed`는 어떤 종류의 도구인가?
2. `sed 's/a/b/'`와 `sed 's/a/b/g'`의 차이는 무엇인가?
3. `-i` 옵션이 위험할 수 있는 이유는 무엇인가?
4. `sed -n '2p'`에서 `-n`과 `p`의 역할은 무엇인가?
5. `diff` 출력의 `<`, `>`, `c`는 무엇을 의미하는가?
6. `sed` 명령이 에러 없이 끝났는데 파일이 바뀌지 않을 수 있는 이유는 무엇인가?
7. 운영 서버 설정 변경 시 파일 수정 후 어떤 검증 단계가 추가로 필요한가?
