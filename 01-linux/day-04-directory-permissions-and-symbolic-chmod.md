# Day 4-2 — 디렉터리 권한과 문자 방식 `chmod`

> 파일과 디렉터리에 표시되는 `rwx`는 같은 글자지만 **의미가 다르다.** 특히 디렉터리의 `x`는 execute라기보다 **traverse/search**, `w`는 내부 Directory Entry 변경과 연결된다는 점을 이해하고 문자 방식 `chmod`를 학습했다.

## 📌 이번에 배운 내용

- 파일과 디렉터리 `rwx` 의미 차이
- Directory Entry란 무엇인가
- 디렉터리 `r`, `w`, `x`
- 경로 접근과 상위 디렉터리 `x`
- 파일 삭제 권한이 파일 자체보다 부모 디렉터리에 크게 좌우되는 이유
- 문자 방식 `chmod`
- `u`, `g`, `o`, `a`, `+`, `-`, `=`
- 숫자 방식과 문자 방식 비교

## 📚 목차

1. Directory란 무엇인가
2. Directory Entry
3. 디렉터리 `rwx`
4. 경로 탐색과 `x`
5. 파일 삭제와 부모 Directory
6. 문자 방식 `chmod`
7. 숫자/문자 방식 비교
8. 실제 실습
9. 헷갈리기 쉬운 부분
10. Troubleshooting
11. 실무 포인트
12. 핵심 정리
13. 복습 문제

## ⚡ 명령어 빠른 복습

| 명령어 | 표현 의미 | 목적 |
|---|---|---|
| `ls -ld dir` | `-l` long, `-d` directory itself | Directory 자체 Permission 확인 |
| `chmod u+x file` | `u` owner, `+` add, `x` execute | Owner에게 x 추가 |
| `chmod g-w dir` | `g` group, `-` remove, `w` write | Group의 write 제거 |
| `chmod o-r file` | `o` others | Others read 제거 |
| `chmod a+r file` | `a` all | 모든 class에 read 추가 |
| `chmod g=rx dir` | `=` exact set | Group 권한을 정확히 `r-x`로 설정 |

---

## 1. Directory란 무엇인가

Directory는 단순한 “파일 상자”가 아니다. 파일 이름과 실제 파일 객체(inode)를 연결하는 **Directory Entry 목록을 관리하는 특별한 파일 타입**으로 이해할 수 있다.

예:

```text
team/
├─ a.txt
└─ b.txt
```

Directory는 내부적으로 `a.txt`, `b.txt` 같은 이름과 해당 객체를 연결하는 정보를 가진다.

이 때문에 Directory의 `w` Permission은 “파일 내용 쓰기”가 아니라 **Directory Entry를 변경하는 권한**과 연결된다.

---

## 2. Directory Entry

쉽게 말하면 Directory 안의 이름 항목이다.

```text
a.txt → 특정 inode
b.txt → 특정 inode
```

새 파일 생성:

```bash
touch team/c.txt
```

은 `team` Directory에 새로운 Entry를 만드는 작업이다.

파일 삭제:

```bash
rm team/a.txt
```

은 `team` Directory의 `a.txt` Entry를 제거하는 작업과 연결된다.

그래서 파일 삭제 권한을 이해할 때 **파일 자체의 write bit만 보면 안 된다.**

---

## 3. 디렉터리의 `r`, `w`, `x`

| 권한 | 파일에서 | 디렉터리에서 |
|---|---|---|
| `r` | 내용 읽기 | Directory Entry 이름 목록 조회 |
| `w` | 내용 수정 | Entry 생성·삭제·rename |
| `x` | 실행 | Directory traverse/search, 내부 경로 접근 |

### `r` — read

```bash
ls directory
```

Directory 안에 어떤 이름이 있는지 목록을 읽는 데 관련된다.

### `w` — write

다음 작업과 연결된다.

```text
파일 생성
파일 삭제
파일 rename
하위 Directory 생성/삭제
```

실제 동작에는 보통 `x`도 함께 필요하다.

### `x` — traverse/search

Directory에서는 `x`를 **경로를 통과할 수 있는 권한**으로 이해한다.

```bash
cd directory
cat directory/file.txt
```

처럼 내부 객체를 실제로 찾아 접근하려면 `x`가 중요하다.

---

## 4. 경로 탐색과 `x`

예:

```text
/var/www/app/config.conf
```

`config.conf`까지 가려면:

```text
/
→ /var
→ /var/www
→ /var/www/app
→ config.conf
```

각 Directory를 통과해야 한다.

따라서 파일 자체가 `644`여도 중간 Directory의 `x` Permission이 없으면 접근이 실패할 수 있다.

```text
파일 read O
부모 directory traverse X
→ 최종 파일 접근 실패 가능
```

이게 `Permission denied` 분석에서 매우 중요한 이유다.

---

## 5. 파일 삭제와 부모 Directory

초심자가 가장 많이 헷갈리는 부분이다.

파일이:

```text
-r--r--r-- file.txt
```

처럼 write Permission이 없어도 **부모 Directory에 적절한 `w+x`가 있으면 파일 삭제가 가능할 수 있다.**

왜냐하면 삭제는 파일 내용 수정이 아니라 부모 Directory의 Entry 제거 작업이기 때문이다.

반대로 파일 자체가 `rw-`여도 부모 Directory에 write Permission이 없으면 rename/delete가 제한될 수 있다.

```text
파일 내용 변경 → 파일 w
파일 이름 생성/삭제/rename → 부모 Directory w(+x)
```

이 개념은 Sticky Bit에서 다시 중요해진다.

---

## 6. 문자 방식 `chmod`

기본 구조:

```text
chmod [대상][연산자][권한] 대상파일
```

### 대상 class

```text
u = user = Owner
g = group
o = others
a = all = u+g+o
```

### 연산자

```text
+ = 기존 권한에 추가
- = 기존 권한에서 제거
= = 지정한 권한으로 정확히 설정
```

### Permission

```text
r = read
w = write
x = execute/traverse
```

예:

```bash
chmod u+x script.sh
```

Owner에게 x만 추가한다.

```bash
chmod g-w shared
```

Group write만 제거한다.

```bash
chmod g=rx shared
```

Group permission을 정확히 `r-x`로 만든다. 기존 Group write가 있었다면 제거된다.

---

## 7. 숫자 방식과 문자 방식 비교

### 숫자 방식

```bash
chmod 750 shared
```

최종 상태 전체를 명확하게 지정할 때 좋다.

```text
7 = rwx
5 = r-x
0 = ---
```

### 문자 방식

```bash
chmod g-w shared
```

기존 상태를 유지하면서 일부 bit만 바꿀 때 좋다.

| 상황 | 적합한 방식 |
|---|---|
| 최종 Permission을 명확히 알고 있음 | 숫자 방식 |
| 기존 Permission 일부만 조정 | 문자 방식 |

---

## 8. 🧪 실제 실습

```bash
mkdir permission-dir
touch permission-dir/file1.txt
ls -ld permission-dir
```

숫자 방식:

```bash
chmod 700 permission-dir
ls -ld permission-dir
chmod 755 permission-dir
ls -ld permission-dir
```

문자 방식:

```bash
chmod u+x file.txt
chmod g-w permission-dir
chmod o-r file.txt
```

각 단계에서 `ls -l` 또는 `ls -ld`로 결과를 검증했다.

---

## 9. ⚠️ 헷갈리기 쉬운 부분

> ⚠️ Directory의 `x`는 프로그램 실행 권한이 아니라 traverse/search 의미가 핵심이다.

> ⚠️ Directory의 `r`은 파일 생성/삭제 권한이 아니다. 이름 목록 조회와 연결된다.

> ⚠️ 파일 삭제 여부는 파일 자체 `w`보다 부모 Directory Permission에 크게 좌우된다.

> ⚠️ `chmod g=rx`는 `r`, `x`를 추가만 하는 것이 아니라 Group Permission을 정확히 그 상태로 만든다.

---

## 10. 🔧 Troubleshooting

### 증상

Directory Permission:

```text
drw-r--r-- shared
```

사용자가:

```bash
cd shared
```

실행했지만 `Permission denied`.

### 가설

Directory traverse `x`가 없다.

### 확인

```bash
id
ls -ld shared
```

### 원인

Directory에서 `x`가 없으므로 해당 경로를 traverse할 수 없다.

### 조치

실제 요구 대상에만 `x`를 추가한다.

```bash
chmod u+x shared
```

### 검증

```bash
ls -ld shared
cd shared
pwd
```

---

## 11. 💼 실무 포인트

경로 Permission 문제는 최종 파일만 보지 않는다.

```bash
namei -l /var/www/app/config.conf
```

같은 도구를 나중에 사용하면 경로의 각 component Permission을 한 번에 확인할 수도 있다. 현재 단계에서는 `ls -ld`로 한 단계씩 확인하는 습관을 먼저 익힌다.

운영 흐름:

```text
현재 사용자/그룹 확인
→ 파일 Permission 확인
→ 부모 Directory Permission 확인
→ 필요한 동작이 read/write/delete/traverse 중 무엇인지 구분
→ 최소한의 bit만 수정
→ 실제 작업 재검증
```

---

## 12. ✅ 핵심 정리

- Directory는 이름→파일 객체 연결 정보를 가진다.
- Directory `r` = 목록 조회, `w` = Entry 변경, `x` = traverse/search.
- 파일 접근에는 상위 Directory `x`가 중요하다.
- 파일 삭제/rename은 부모 Directory Permission과 밀접하다.
- 문자 `chmod`는 `u/g/o/a + - = r/w/x` 조합이다.
- `=`는 해당 class의 Permission을 정확히 지정한다.

---

## 13. 🧠 복습 문제

1. Directory Entry란 무엇인가?
2. 파일과 Directory에서 `rwx` 의미를 비교해보라.
3. 왜 파일이 `644`여도 접근이 실패할 수 있는가?
4. 왜 read-only 파일도 삭제될 수 있는가?
5. `chmod g-w dir`를 해석해보라.
6. `chmod g=rx dir`와 `chmod g+rx dir`의 차이는?
7. Directory에서 `w`만 있고 `x`가 없으면 왜 실용적으로 작업이 제한되는가?
