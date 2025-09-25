.PHONY: build run
build:
	GOOS=wasip1 GOARCH=wasm go build -buildmode=c-shared -o main.wasm ./
run: build
	envoy -c ./envoy.yaml
