FROM golang:latest

WORKDIR $GOPATH/src/hellokube
COPY .  $GOPATH/src/hellokube
RUN go build .

EXPOSE 8080
ENTRYPOINT [ "./hellokube" ]