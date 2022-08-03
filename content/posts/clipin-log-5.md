+++
title = 'Clipin Log 5. 单屏幕Pin实现'
date = 2021-01-09 21:41:18
+++

Pin功能的触发时机是在ClipManager的select状态下点击回车时，所以在ClipWindowController的done中，将bitmapRep和对应的rect作为参数，执行Pin操作

```swift
class ClipWindowController: NSWindowController {
	// ....
    @objc func done() {
        guard let view = self.clipView,
              let rect = view.drawingRect
        else {
            return
        }
        guard let bitmapRep = view.bitmapImageRepForCachingDisplay(in: rect) else {return}
        view.cacheDisplay(in: rect, to: bitmapRep)

        // 利用bitmapRep进行Pin操作
        PinManager.shared.pin(rep: bitmapRep, rect: rect)
    }

	// ....

}
```

## PinManager单例

主要完成将图像作为一个新的窗口，并放置在屏幕顶层的操作，这里称为pin操作。源码里还实现了一些保存图像的函数，后续会在小工具中使用。

pin函数初始化了页面、窗口和控制器并显示窗口，在处理结束后发送`pinEnd`通知。

```swift
class PinManager: NSObject {
    static let shared = PinManager()
    var controllers: [PinWindowController] = []
    
    func pin(rep: NSBitmapImageRep, rect:NSRect) {
        let image = NSImage(size: rect.size)
        image.addRepresentation(rep)
        let view = PinView(image: image)
        let window = PinWindow(rect: rect, contentView: view)
        let controller = PinWindowController(window: window)
        self.controllers.append(controller)
        controller.showWindow(nil)
        NotificationCenter.default.post(name: NotiNames.pinEnd.name, object: nil)
    }
}
```

## 图像显示到窗口中

和ClipView类似，在PinView的中渲染图像：

```swift
class PinView: NSView {
    
    var image: NSImage?
    
    init(image: NSImage) {
        super.init(frame: .zero)
        self.image = image
    }

    required init?(coder: NSCoder) {
        super.init(coder: coder)
    }
    
    override func draw(_ dirtyRect: NSRect) {
        super.draw(dirtyRect)

        // Drawing code here.
        guard let image = self.image else { return }
        image.draw(in: self.frame, from: self.frame, operation: .sourceOver, fraction: 1.0)
    }
    
}
```

## 窗口细节特性设置

Pin窗口包含以下特性：

- 无边框
- 在屏幕顶层
- 点击窗口内部可以实现拖动
- 鼠标在窗口内时显示透明标题栏（左上角的三个按钮），移除窗口标题栏消失

上述功能都可以通过PinWindowController和PinWindow的初始化配置实现

```swift
class PinWindowController: NSWindowController {

    var pinWindow: PinWindow?
    
    init(window: PinWindow) {
        super.init(window: window)
        self.pinWindow = window
        guard let window = self.window, let view = window.contentView else { return }
				// 添加鼠标进入或退出的监听
        let trackingArea = NSTrackingArea(rect: view.frame, options: [.activeAlways, .mouseEnteredAndExited], owner: self, userInfo: [:])
        view.addTrackingArea(trackingArea)
        
    }

		// .... some functions
        
    override func mouseEntered(with event: NSEvent) {
        guard let window = self.pinWindow else { return }
        window.showTitle()
    }
    
    override func mouseExited(with event: NSEvent) {
        guard let window = self.pinWindow else { return }
        window.hideTitle()
    }
        
}
```

```swift
class PinWindow: NSWindow {
    init(rect: NSRect, contentView: PinView) {
        super.init(contentRect: rect, styleMask: [.closable, .titled, .miniaturizable, .fullSizeContentView], backing: .buffered, defer: false)
        self.titleVisibility = .visible
        self.titlebarAppearsTransparent = true
        self.contentView = contentView
        self.level = .floating  // 改变窗口显示等级
        self.isMovableByWindowBackground = true  // 可以通过拖拽窗口内容移动
        NotificationCenter.default.addObserver(self, selector: #selector(pinFloating), name: NotiNames.pinFloating.name, object: nil)
        NotificationCenter.default.addObserver(self, selector: #selector(pinNormal), name: NotiNames.pinNormal.name, object: nil)
    }
    
    func hideTitle() {
        self.styleMask = [.titled, .fullSizeContentView]  // 隐藏标题栏，原理详见官方文档
    }
    
    func showTitle() {
        self.styleMask = [.closable, .titled, .miniaturizable, .fullSizeContentView]  // 显示标题栏，原理详见官方文档
    }
    
    @objc func pinFloating() {
        self.level = .floating
    }
    
    @objc func pinNormal() {
        self.level = .normal
    }    
    
}
```

代码中还让PinWindow添加两个消息`pinFloating`和`pinNormal`的监听，是为了在有Pin窗口的前提下，使用Clip功能时不让Pin窗口遮挡住Clip窗口，所以两个消息分别在`ClipManager.shared.start()`和`ClipManager.shared.end()`中添加：

```swift
class ClipManager {

	// ....
    
    func start() {
        NotificationCenter.default.post(name: NotiNames.pinNormal.name, object: nil)
        // ....
    }
    
    @objc func end() {
        NotificationCenter.default.post(name: NotiNames.pinFloating.name, object: nil)
        // ....
    }
    
}
```
接下来是[多屏幕支持](https://hagemon.github.io/post/clipin-post-6/)