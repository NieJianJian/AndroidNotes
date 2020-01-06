### 部分系统服务及其作用

| 引导服务                    | 作用                                                         |
| :-------------------------- | ------------------------------------------------------------ |
| Installer                   | 系统安装APK时的一个服务类，启动完成Installer服务之后才能启动其他的系统服务 |
| ActivityManagerService      | 负责四大组件的启动、切换、调度                               |
| PowerManagerService         | 计算系统中的Power相关的计算，然后决策系统应该如何反映        |
| LightsService               | 管理和显示背光LED                                            |
| DisplayManagerService       | 用来管理所有显示设备                                         |
| UserManagerService          | 多用户模式管理                                               |
| SensorService               | 为系统提供各种感应器服务                                     |
| PackageManagerService       | 用来对APK进行安装、解析、删除、卸载等操作                    |
| ... ...                     |                                                              |
| **核心服务**                |                                                              |
| DropBoxManagerService       | 用于生成和管理系统运行时的一些日志文件                       |
| BatteryService              | 管理电池相关的服务                                           |
| UsageStatsService           | 收集用户使用每一个App的帧率、使用时长                        |
| WebViewUpdateService        | WebView服务                                                  |
| **其他服务**                |                                                              |
| CameraService               | 摄像头相关服务                                               |
| AlarmManagerService         | 全局定时器相关服务                                           |
| InputManagerService         | 管理输入事件                                                 |
| WindowManagerService        | 窗口管理服务                                                 |
| VrManagerService            | VR模式管理服务                                               |
| BluetoothService            | 蓝牙管理服务                                                 |
| NotificationManagerService  | 通知管理服务                                                 |
| DeviceStorageMonitorService | 存储相关管理服务                                             |
| LocationManagerService      | 定位管理服务                                                 |
| AudioService                | 音频相关管理服务                                             |
| ... ...                     |                                                              |

