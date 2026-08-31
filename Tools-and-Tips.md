## 학습 도구 및 기타 팁
### Markdown
markdown .md 파일은 줄 바꿈을 하려면 문장 끝에 공백 2개(enter나 space)를 넣거나 br 사용하기.

### Markdown 미리보기
VS Code에서 `Ctrl + Shift + V`를 누르면 Markdown 미리보기가 열려서 GitHub에 어떻게 표시될지 업로드 전에 확인 가능.

### Bash Here Document로 여러 줄 파일 만들기
여러 줄의 내용을 한 번에 파일에 저장할 때 `cat`과 Here Document(`<<`)를 함께 사용할 수 있다.

```bash
cat > access.log <<'EOF'
10.0.0.1 GET /index.html 200 120
10.0.0.2 POST /login 401 80
10.0.0.3 GET /login 500 150
EOF
```

동작 흐름:

```text
cat
→ 표준 입력을 받음

<<'EOF'
→ EOF가 나올 때까지 여러 줄을 입력으로 전달

> access.log
→ 전달받은 내용을 access.log에 저장
```

`EOF`는 입력 종료를 표시하는 구분자이며 반드시 `EOF`라는 이름일 필요는 없다.

```bash
cat > test.txt <<'END'
hello
linux
server
END
```

`>`는 기존 파일을 덮어쓰고, `>>`는 기존 내용 뒤에 추가한다.

```bash
cat >> test.txt <<'EOF'
new line 1
new line 2
EOF
```

정리:

```text
cat > file <<'EOF'   → 여러 줄 입력 후 파일 덮어쓰기
cat >> file <<'EOF'  → 여러 줄 입력 후 기존 파일 뒤에 추가
```

실습용 로그 파일이나 설정 샘플을 여러 줄로 빠르게 만들 때 유용하다.

