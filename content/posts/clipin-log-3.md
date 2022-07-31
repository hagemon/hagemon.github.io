---
title: 'Clipin Log 3. 截屏页面的实现'
date: 2021-01-09 21:24:06
tags: [iOS,Clipin]
published: true
hideInList: false
feature: 
isTop: false
---
## 实现思路

- 创建和屏幕大小相等的NSWindow，且拥有以下特性：
    - borderless
    - contentview透明背景
    - level在其余窗口之上
- 捕捉高亮区域
- 重载`NSWindow.contentView`中的draw方法，渲染高亮区域的图像覆盖背景
- 按下ESC退出Clip
- 按下Enter完成对高亮区域图像的获取
- 用NSWindowController控制以上行为

## 实现过程

创建NSWindowController，NSWindow，NSView对应的子类。

在`ClipManager.shared.start()`中创建控制器、窗口和页面的对象并建立关系：

```swift
func start() {
	guard let screen = NSScreen.main else { return }  // 单显示器环境
	NSApplication.shared.activate(ignoringOtherApps: true)  // 在触发快捷键后激活App，使开启的窗口能够变为key window
    let view = ClipView(frame: screen.frame)
    let clipWindow = ClipWindow(contentRect: screen.frame, contentView: view)
    let clipWindowController = ClipWindowController(window: clipWindow)
    self.controllers.append(clipWindowController) // 将其加入当前保存的控制器中
    self.status = .ready
    clipWindowController.capture(screen)
}
```

ClipWindow的条件在初始化时既可以满足：

```swift
class ClipWindow: NSWindow {
    
    init(contentRect: NSRect, contentView: ClipView) {
        super.init(contentRect: contentRect, styleMask: .borderless, backing: .buffered, defer: false)
        self.contentView = contentView // ClipView as its contentView
        self.level = .statusBar
    }
}
```

我们暂时不使用鼠标监听事件来获取高亮区域，而是使用一个临时的NSRect来表示我们所选取的范围：

```swift
let tmpRect = NSRect(x: 300, y: 300, width: 200, height: 400)
```

调用`ClipWindowController.capture(screen)`进入准备Clip的状态。在capture中实现获取当前屏幕图像，并设置Window的透明背景功能，最后显示ClipWindow：

```swift
class ClipWindowController: NSWindowController {
		var clipView: ClipView?  // 为了方便使用self.window.view
		var screenImage: NSImage?  // 背景图片
		let tmpRect = NSRect(x: 300, y: 300, width: 200, height: 400)

		func capture(screen: NSScreen) {
            guard let window = self.window else { return }
            guard let cgScreenImage = CGDisplayCreateImage(CGMainDisplayID())                     else { return } // 获取主显示器的CGImage
            self.screenImage = NSImage(cgImage: cgScreenImage, size: screen.frame.size) // 转化为NSImage
            window.backgroundColor = NSColor(white: 0, alpha: 0.5) // 设置背景为透明，透明度为0.5
            self.clipView = window.contentView as? ClipView
            self.showWindow(nil)
		}
}
```

渲染高亮区域图像是最重要的步骤，通过重载ClipView的draw函数实现渲染功能：

```swift
class ClipView: NSView {
    
    var image: NSImage?  //屏幕图像
    var drawingRect: NSRect?  // 高亮区域

    override func draw(_ dirtyRect: NSRect) {
        
        super.draw(dirtyRect)

        // Drawing code here.
        guard let image = self.image, let drawingRect = self.drawingRect else {
            return
        }
        var rect = NSIntersectionRect(drawingRect, self.bounds)  // 取高亮区域和页面的交集区域
        rect = NSIntegralRect(rect)  // 区域取整，为了防止抖动问题

				// 取屏幕图像中高亮区域部分渲染在view上，使用.sourceOver操作
				// 由于source图像是alpha为1，所以渲染结果即是区域图像本身
        image.draw(in: rect, from: rect, operation: .sourceOver, fraction: 1.0)
    }    
}
```

其中image和drawingRect变量会在绘制的过程中动态的给出，Controller中的`highlight()`函数会获取当前高亮区域的信息并告知系统view需要刷新：

```swift
// In ClipWindowController
func highlight() {
    guard let rect = self.tmpRect, // 暂时使用临时区域，之后使用
          // let rect = self.highlightRect,
					let image = self.screenImage,
          let view = self.clipView
    else { return }
    DispatchQueue.main.async {
        if view.image == nil {
            view.image = image
        }
        view.drawingRect = rect  // 动态改变绘制区域的值
        view.needsDisplay = true  // 告知系统view需要刷新
    }
}
```

## ESC退出Clip

之前提到，在AppDetegate中注册了局部事件的监听，在捕捉到ESC后执行单例的结束操作`ClipManager.shared.end()` ，对当前开启的所有controller的窗口进行关闭操作：

```swift
class ClipManager {
    
    static let shared = ClipManager() // 单例
    var status: ClipStatus = .off // Clip初始状态
		var controllers: [ClipWindowController] = []  // 保存当前打开的所有控制器
    
		// .... other functions
    
    @objc func end() {
        for controller in self.controllers {
            controller.window?.orderOut(nil)
        }
        self.controllers.removeAll()
        self.status = .off 
    }
    
}
```

## Enter获取高亮区域图像

之前提到，在Enter后发送`clipEnd`的通知，在ClipWindowController中捕捉：

```swift
class ClipWindowController: NSWindowController {

    // .... variables
    
    func capture(_ screen:NSScreen) {
        NotificationCenter.default.addObserver(self, selector: #selector(self.done), name: NotiNames.clipEnd.name, object: nil)
				// .... Get screen image
    }

    @objc func done() {
        guard let view = self.clipView,
              let rect = view.drawingRect
        else {
            return
        }
				// 获取view中特定rect对于的bitmapRep
        guard let bitmapRep = view.bitmapImageRepForCachingDisplay(in: rect) else {return}
        view.cacheDisplay(in: rect, to: bitmapRep)

        // bitmapRep即为获取到的图像信息，可以进行保存或别的操作

        // 利用bitmapRep进行Pin操作，暂不实现
        // ....
    }

    // .... other functions

}
```
接下来是[高亮区域的选取](https://hagemon.github.io/post/clipin-log-4/)