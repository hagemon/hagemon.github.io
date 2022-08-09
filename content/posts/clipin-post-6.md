+++
title = 'Clipin Log 6. 多屏幕支持'
date = 2021-01-09 21:44:43
tags = ['iOS dev']
+++

## 多屏幕环境

在多屏幕环境下，各个屏幕的起始坐标`origin`会根据用户在系统偏好设置中对显示器排列的设置而不同，下图直观地表示了多屏幕环境下各屏幕的坐标。

![](https://hagemon.github.io/post-images/1613224003347.png)

通过`screen.visibleFrame`可以访问屏幕在此坐标系下的frame。

## 实现思路

- 在ClipManager.shared.start()中检测所有屏幕并创建控制器
- 对于每个控制器，获取对应显示器当前页面图像
- 在执行Pin操作时，需要将窗口偏移到正确的位置

```swift
class ClipManager {
	// ....    
    var controllers: [ClipWindowController] = []
    // ....
    func start() {
        NotificationCenter.default.post(name: NotiNames.pinNormal.name, object: nil)
        NSApplication.shared.activate(ignoringOtherApps: true)
        
        for screen in NSScreen.screens {  // 获取所有屏幕
            let view = ClipView(frame: screen.frame)
            let clipWindow = ClipWindow(contentRect: screen.frame, contentView: view)
            let clipWindowController = ClipWindowController(window: clipWindow)
            controllers.append(clipWindowController)
            self.status = .ready
            clipWindowController.capture(screen)
        }

    }
    
    // ....
}
```

在ClipWindowController中获取各屏幕的图像：

```swift
class ClipWindowController: NSWindowController {
	func capture(_ screen:NSScreen) {
        guard let window = self.window else { return }
        //  获取screen对应的displayID和ID对应的图像
        guard let displayID = screen.deviceDescription[NSDeviceDescriptionKey(rawValue: "NSScreenNumber")] as? CGDirectDisplayID,
                let cgScreenImage = CGDisplayCreateImage(displayID)
        else { return }
        self.screenImage = NSImage(cgImage: cgScreenImage, size: screen.frame.size)
        window.backgroundColor = NSColor(white: 0, alpha: 0.5)
        self.clipView = window.contentView as? ClipView
        self.showWindow(nil)
    }
}
```

在PinManager中修改pin函数，将屏幕起始坐标增加为新的参数，并对pin窗口进行偏移：

```swift
class PinManager: NSObject {
	func pin(rep: NSBitmapImageRep, rect:NSRect, screenOrigin:NSPoint) {
        let image = NSImage(size: rect.size)
        image.addRepresentation(rep)
        let view = PinView(image: image)
        let window = PinWindow(rect: rect, contentView: view)
        let controller = PinWindowController(window: window)
        self.controllers.append(controller)
        // 以屏幕左下角坐标为基准，偏移pin窗口
        window.setFrameOrigin(screenOrigin.offsetBy(dx: rect.origin.x, dy: rect.origin.y))
        controller.showWindow(nil)
        NotificationCenter.default.post(name: NotiNames.pinEnd.name, object: nil)
    }
}
```

同时在调用pin函数时增加对屏幕起始坐标参数的传入：

```swift
// ClipWindowController.swift
@objc func done() {
    guard let view = self.clipView,
          let rect = view.drawingRect
    else {
        return
    }
    guard let bitmapRep = view.bitmapImageRepForCachingDisplay(in: rect),
          let window = self.window,
          let screen = window.screen
    else {return}
    view.cacheDisplay(in: rect, to: bitmapRep)
    PinManager.shared.pin(rep: bitmapRep, rect: rect, screenOrigin: screen.visibleFrame.origin)
}
```

接下来是[对截屏的一些优化](https://hagemon.github.io/post/clipin-post-7/)