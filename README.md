# iotserver-sdk-go

iotserver 物联网平台的 **驱动 SDK**。协议驱动基于它开发，无需自己处理 gRPC 通信、
配置加载、设备缓存等底层细节。

## 它解决什么

写一个协议驱动需要：读平台下发的配置、起 gRPC 服务端（供平台回调）、
连平台 gRPC（上报数据）、维护设备缓存、处理日志与优雅退出。
这些都由 SDK 封装，驱动只需实现 `interfaces.Driver` 接口，专注协议本身。

```go
package main

import (
    "github.com/LMgateways/iotserver-sdk-go/commons"
    "github.com/LMgateways/iotserver-sdk-go/service"
)

func main() {
    // 构造：读配置 + 连平台 + 拉取本实例的设备列表
    sd := service.NewDriverService("my-driver", commons.HummingbirdIot)

    // 你自己的驱动实现（见 interfaces.Driver）
    driver := NewMyDriver(sd)

    // 启动：起 gRPC 服务端、注册回调、阻塞等待退出信号
    if err := sd.Start(driver); err != nil {
        sd.GetLogger().Error("start failed: %s", err)
    }
}
```

## 驱动要实现什么

```go
type Driver interface {
    CloudPluginNotify(ctx, t commons.CloudPluginNotifyType, name string) error
    DeviceNotify(ctx, t commons.DeviceNotifyType, deviceId string, device model.Device) error
    ProductNotify(ctx, t commons.ProductNotifyType, productId string, product model.Product) error
    Stop(ctx) error
    HandlePropertySet(ctx, deviceId string, data model.PropertySet) error
    HandlePropertyGet(ctx, deviceId string, data model.PropertyGet) error
    HandleServiceExecute(ctx, deviceId string, data model.ServiceExecuteRequest) error
}
```

## 常用 API

```go
// 数据上报
sd.PropertyReport(deviceId, model.NewPropertyReport(false, map[string]model.PropertyData{
    "temp": model.NewPropertyData(23.5),
}))
sd.EventReport(deviceId, model.NewEventReport(false, model.NewEventData("alarm", params)))

// 设备状态
sd.Online(deviceId)
sd.Offline(deviceId)
sd.GetConnectStatus(deviceId)

// 设备与物模型
sd.GetDeviceList()
sd.GetDeviceById(deviceId)
sd.GetProductProperties(productId)
sd.CreateDevice(model.NewAddDevice(...))

// 下发响应
sd.PropertySetResponse(deviceId, ...)
sd.ServiceExecuteResponse(deviceId, ...)

// 持久化
sd.PutCustomStorage(map[string][]byte{...})
sd.GetCustomStorage([]string{"key"})

// 日志
sd.GetLogger().Infof("...")

// 自定义参数（平台驱动实例编辑页里填的 JSON）
sd.GetCustomParam()
```

## 依赖

- `github.com/LMgateways/iotserver-driver-proto` — gRPC 协议定义
- `github.com/eclipse/paho.mqtt.golang` — MQTT 客户端（部分驱动用）
- `go.uber.org/zap` — 日志

## 与上游的差异

fork 自 `github.com/winc-link/hummingbird-sdk-go`，改动：

- 模块路径改为 `github.com/LMgateways/iotserver-sdk-go`；
- 依赖从 `winc-link/edge-driver-proto` 换为 `LMgateways/iotserver-driver-proto`；
- 升级 grpc / protobuf 依赖版本。

**API 与上游一致**，驱动代码从上游 SDK 迁移过来只需改 import 路径。

## 参考实现

本组织下的三个协议驱动即基于本 SDK：

- [iotserver-mqtt-driver](https://github.com/LMgateways/iotserver-mqtt-driver)
- [iotserver-tcp-driver](https://github.com/LMgateways/iotserver-tcp-driver)
- [iotserver-http-driver](https://github.com/LMgateways/iotserver-http-driver)