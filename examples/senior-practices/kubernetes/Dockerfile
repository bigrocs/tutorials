FROM bigrocs/golang-gcc:1.13 as builder

WORKDIR /go/src/github.com/bigrocs/tutorials/examples/senior-practices/kubernetes
COPY . .

ENV CGO_ENABLED=1 GOOS=linux GOARCH=amd64
RUN go build -a -installsuffix cgo -o bin/service

FROM bigrocs/alpine:ca-data

RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

COPY --from=builder /go/src//go/src/github.com/bigrocs/tutorials/examples/senior-practices/kubernetes/bin/service /usr/local/bin/
CMD ["service"]