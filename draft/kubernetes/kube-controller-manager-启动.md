
### 启动流程

* 解析命令行，生成配置相关的信息
* 开始执行，主要的函数在cmd/kube-controller-manager/app/controllermanager.go:176。 
* 开启事件处理流水线。接收事件然后发送到EventSink处理。
* configz.Config
* 一些安全接口