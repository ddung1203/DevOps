# Docker

## Docker Engine 설치

```bash
sudo apt install ca-certificates curl gnupg lsb-release
```

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```bash
sudo apt update
```

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

```bash
sudo usermod -aG docker ubuntu
sudo usermod -aG docker jenkins
```

## 멀티스테이지 빌드
컨테이너 이미지를 만들면서 빌드 등에는 필요하지만, 최종 컨테이너 이미지에는 필요없는 환경을 제거할 수 있도록 단계를 나누어 이미지를 만드는 방법이다.

![DevOps](../images/docker_1.png)

멀티스테이지 빌드를 사용하게 되면 상기의 그림처럼 컨테이너 실행 시에는 빌드에 사용한 파일 및 디렉터리와 같은 의존 파일들이 모두 삭제된 상태로 컨테이너가 실행되게 된다. 결론적으로 더 가벼운 크기의 컨테이너를 사용할 수 있게 된다.

```
FROM golang:1.7.3 AS builder
WORKDIR /go/src/github.com/ddung1203/hello-world/
RUN go get -d -v golang.org/x/net/html
COPY app.go    .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /go/src/github.com/ddung1203/hello-world/app .
CMD ["./app"]
```
