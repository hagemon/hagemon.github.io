+++
title = 'Clipin Log 2. 截屏管理器'
date = 2021-01-09 21:13:22
+++

Manager/ClipManager.swift

## 截屏状态

```swift
enum ClipStatus {
    case off     // 未开启
    case ready   // 就绪
    case start   // 开始选择区域
    case select  // 已选择区域
    case adjust  // 调整区域大小
    case drag    // 拖拽区域
}
```

截屏的状态转移可以表述为下图

![](https://hagemon.github.io/post-images/1613222166438.png)

## 单例

使用单例来控制Clip的操作：

```swift
import Cocoa

class ClipManager {
    
    static let shared = ClipManager() // 单例
    var status: ClipStatus = .off // Clip初始状态
	var controllers: [ClipWindowController] = []  // 保存当前打开的所有控制器
    
    private init() {
        NotificationCenter.default.addObserver(self, selector: #selector(self.end), name: NotiNames.pinEnd.name, object: nil)
    }
    
    func start() {
        // 开始Clip
    }
    
    @objc func end() {
        // 结束Clip
    }
    
}
```

通过调用`ClipManager.shared.start()`进入Clip功能。

这里提前提一下，在初始化单例时添加对结束通知的监听，当某个组件完成了Pin的工作，则会post对应的通知，在ClipManager中处理Pin完成之后的操作：

```swift
private init() {
		NotificationCenter.default.addObserver(self,
                                           selector: #selector(self.end),
                                           name: NotiNames.pinEnd.name,
                                           object: nil)
}

@objc func end() {
    // Clip结束后的操作
}
```

此处的notification name是枚举变量：

```swift
enum NotiNames: String {
    var name: NSNotification.Name {
        return Notification.Name(self.rawValue)
    }
    case clipEnd = "doEndClip"  // 结束Clip
    case pinEnd = "doEndPin"  // 结束Pin
    case pinFloating = "pinFloating"  // Pin窗口上浮
    case pinNormal = "pinNormal"  // Pin窗口下沉
}
```

## Clip操作相关的通知传递

对于Clip操作，在select状态下按下Enter会发送`clipEnd`的通知，由WindowController接收通知并做后续处理，在之后的章节中会详细介绍。

接下来进入[截屏界面的实现](https://hagemon.github.io/post/clipin-log-3)