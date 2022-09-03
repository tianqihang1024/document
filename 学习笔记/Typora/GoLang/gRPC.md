





## 创建`GoLang`项目

-   创建文件夹，使用`GoLand`打开，在`terminal`窗口执行下面操作
-   使用`go env -w GO111MODULE=on`管理`go`项目依赖
-   使用`go mod init 项目名称`初始化项目



## 准备`gRPC`环境

- 下载依赖

  ```shell
  go env -w GOPROXY=https://goproxy.cn
  go get google.golang.org/protobuf/cmd/protoc-gen-go
  go get google.golang.org/grpc/cmd/protoc-gen-go-grpc
  
  go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
  go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
  ```

  

-   









protoc monitor.proto --go_out=.\ 







```
// 指定当前proto语法版本
syntax = "proto3";
// option go_package = "path;name"; path 表示生成的go文件存放地址，会自动生成目录
// name 表示生成的go文件所属包名
option go_package="../service";
// 指定文件生成出来的package
package service;

message User {
  string userName = 1;
  int32 age = 17;

}
```

