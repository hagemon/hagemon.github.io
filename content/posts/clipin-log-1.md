+++
title = 'Clipin Log 1. 基于Status Bar的应用&监听快捷键'
date = 2021-01-09 20:53:34
tags = ['iOS dev']
+++

# 创建一个基于Status Bar的应用

基于Status Bar的应用不需要设置额外的UI，所以不需要使用StoryBoard，只要在AppDelegate中配置好相应的StatusBarItem即可。

## Storyboard

由于是基于Status Bar的应用，所以在Storyboard中将页面相关的组件都删除了，保留了Menu相关的组件。

## 配置info.plist

在info.plist中添加名为`Application is agent (UIElement)`的key，并将value值设置为`YES`。这样App就不会在Dock中显示

## AppDelegate

在AppDelegate中创建StatusBar按钮并配置对应的action

```swift
import Cocoa

@main
class AppDelegate: NSObject, NSApplicationDelegate {
    
    let statusItem = NSStatusBar.system.statusItem(withLength:NSStatusItem.squareLength)

    func applicationDidFinishLaunching(_ aNotification: Notification) {        
        guard let button = self.statusItem.button else { return }
        button.image = NSImage(named: NSImage.Name("StatusBarIcon")) // StatusBarIcon is in Assets.xcassets
        button.action = #selector(showMenu)
    }

    func applicationWillTerminate(_ aNotification: Notification) {
        // Insert code here to tear down your application
    }
        
    @objc func showMenu() {//to be do}
}
```

# 监听快捷键

基于HotKey实现对全剧快捷键的监听，HotKey底层使用了`Carbon`的`RegisterEventHotKey`接口：

```swift

static let eventHotKeySignature: UInt32 = {
    let string = "SSHk"
    var result: FourCharCode = 0
    for char in string.utf16 {
        result = (result << 8) + FourCharCode(char)
    }
    return result
}()

// Register with the system
var eventHotKey: EventHotKeyRef?
let hotKeyID = EventHotKeyID(signature: eventHotKeySignature, id: box.carbonHotKeyID)
let registerError = RegisterEventHotKey(
    hotKey.keyCombo.carbonKeyCode,
    hotKey.keyCombo.carbonModifiers,
    hotKeyID,
    GetEventDispatcherTarget(),
    0,
    &eventHotKey
)
```

## 全局快捷键（基于HotKey）

为了启动截屏功能，需要监听全局的快捷键。在AppDelegate中使用HotKey对全局快捷键`cmd+shift+a`进行注册：

```swift
import Cocoa
import HotKey

@main
class AppDelegate: NSObject, NSApplicationDelegate {
    
    let statusItem = NSStatusBar.system.statusItem(withLength:NSStatusItem.squareLength)
    let hotKey = HotKey(key: .a, modifiers: [.shift, .command])

    func applicationDidFinishLaunching(_ aNotification: Notification) {
               
        self.hotKey.keyUpHandler = {
            print("hello")
        }
        
        guard let button = self.statusItem.button else { return }
        button.image = NSImage(named: NSImage.Name("StatusBarIcon"))
        button.action = #selector(showMenu)
    }

    func applicationWillTerminate(_ aNotification: Notification) {
        // Insert code here to tear down your application
    }
    
    @objc func showMenu() {}
}
```

## 局部快捷键

局部快捷键用于监听应用内的一些键盘事件，比如回车完成Clip操作，Esc退出Clip操作：

```swift
import Cocoa
import HotKey

@main
class AppDelegate: NSObject, NSApplicationDelegate {
    
    // ... variables

    func applicationDidFinishLaunching(_ aNotification: Notification) {

        // local hotkey
        NSEvent.addLocalMonitorForEvents(matching: .keyDown, handler: {
            (event) -> NSEvent? in
            if ClipManager.shared.status != .off && event.keyCode == kVK_Escape {
                // 退出Clip之后的操作
                ClipManager.shared.end()
            }
            if ClipManager.shared.status == .select && event.keyCode == kVK_Return {
                // 完成Clip之后的操作，post一个名为clipEnd的通知
                NotificationCenter.default.post(name: NotiNames.clipEnd.name, object: self, userInfo: nil)
            }
            return nil
        })

        // ... global hotkey and status bar item
    }

    func applicationWillTerminate(_ aNotification: Notification) {
        // Insert code here to tear down your application
    }
    
}
```

接下来看看[截屏管理器](https://hagemon.github.io/post/clipin-log-2/)