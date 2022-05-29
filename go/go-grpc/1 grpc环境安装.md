# grpc环境安装

1. 安装protobuf库

   ```shell
   go get github.com/golang/protobuf/proto
   ```

2. 安装grpc库 (vpn)

   ```shell
   go get google.golang.org/grpc
   ```

3. 安装模板生成库

   ```shell
   go get github.com/golang/protobuf/protoc-gen-go
   ```

4. 下载一个protoc二进制执行文件，放置在GOPATH的bin目录下 [下载地址](https://github.com/protocolbuffers/protobuf/releases/tag/v3.9.0)



现在可以使用protobuf定义服务，并自动生成客户端和服务的的功能库了