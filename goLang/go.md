### 설치
```bash
wget https://go.dev/dl/go1.23.2.linux-amd64.tar.gz
# 또는 
curl -L -O https://go.dev/dl/go1.23.2.linux-amd64.tar.gz

# go 바이너리 파일 설치
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.23.2.linux-amd64.tar.gz

# go 실행파일 위치 지정
export PATH=$PATH:/usr/local/go/bin

# go 설치 확인
go version
```

### 패키지 생성
```bash
go mod init 폴더명
```

### 실행
```bash
go run .
```

### 새로운 모듈 추가
```bash
go mod tidy
```

![alt text](https://pangorithm.github.io/MyAuxiliaryStorage/image/go-cheat-sheet.webp)