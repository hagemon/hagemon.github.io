---
title: 'Clipin Log 7. 提升截屏流畅度'
date: 2021-02-24 23:23:36
tags: []
published: true
hideInList: false
feature: 
isTop: false
---
# 旧版本

在之前的版本中，我在一个ClipView上对暗色背景和高亮区域绘制：

``` swift
override func draw(_ dirtyRect: NSRect) {
    
    super.draw(dirtyRect)

    // Drawing code here.
    guard let image = self.image, let drawingRect = self.drawingRect else {
        return
    }
    image.draw(in: self.bounds, operation: .sourceOver, fraction: 0.5) // 绘制背景
    var rect = NSIntersectionRect(drawingRect, self.bounds)  // 取高亮区域和页面的交集区域
    rect = NSIntegralRect(rect)  // 区域取整，为了防止抖动问题

            // 取屏幕图像中高亮区域部分渲染在view上，使用.sourceOver操作
            // 由于source图像是alpha为1，所以渲染结果即是区域图像本身
    image.draw(in: rect, from: rect, operation: .sourceOver, fraction: 1.0)
}    
```

# 新版本

将背景View和高亮区域View分离，在NSWindow中分别添加两个view：

``` swift
import Cocoa

class ClipWindow: NSWindow {
    
    var rectView: RectView
    var backView: BackView
    
    init(contentRect: NSRect, backView: BackView, rectView: RectView) {
        self.rectView = rectView
        self.backView = backView

        super.init(contentRect: contentRect, styleMask: .borderless, backing: .buffered, defer: false)
        self.contentView?.addSubview(backView)
        self.contentView?.addSubview(rectView)
        self.level = .statusBar
    }
    
}
```

## 绘制高亮区域

``` swift
import Cocoa

class RectView: NSView {
    
    var image: NSImage?
    var drawingRect: NSRect?
    var showDots = false
    var paths: [(NSBezierPath, DotType)] = []
    
    override func draw(_ dirtyRect: NSRect) {
        super.draw(dirtyRect)

        // Drawing code here.
        guard let image = self.image,
              let drawingRect = self.drawingRect
        else {
            return
        }
        var rect = NSIntersectionRect(drawingRect, self.bounds)
        rect = NSIntegralRect(rect)
        image.draw(in: rect, from: rect, operation: .sourceOver, fraction: 1.0)
        if self.showDots {
            self.drawDots(rect: rect)
        }
    }
    
    private func drawDots(rect: NSRect) {
        let radius: CGFloat = 1.5
        let dots = self.getDotsCoord(with: rect, radius: radius)
        self.paths = []
        for (dot, type) in dots {
            let path = NSBezierPath(ovalIn: NSRect(origin: dot, size: CGSize(width: radius*2, height: radius*2)))
            self.paths.append((path, type))
            path.lineWidth = 3
            NSColor.white.set()
            path.stroke()
        }
    }
    
    func getDotsCoord(with rect:NSRect, radius: CGFloat) -> [(NSPoint, DotType)] {
        let width = rect.width
        let height = rect.height
        let origin = rect.offsetBy(dx: -radius, dy: -radius).origin
        let dots: [(NSPoint, DotType)] = [
            (origin, .corner),
            (origin.offsetBy(dx: width/2, dy: 0), .bottom),
            (origin.offsetBy(dx: 0, dy: height/2), .left),
            (origin.offsetBy(dx: width, dy: 0), .corner),
            (origin.offsetBy(dx: 0, dy: height), .corner),
            (origin.offsetBy(dx: width, dy: height/2), .right),
            (origin.offsetBy(dx: width/2, dy: height), .top),
            (origin.offsetBy(dx: width, dy: height), .corner)
        ].map({
            (point, type) in
            return (NSPoint(x: Int(point.x), y: Int(point.y)), type)
        })
        return dots
    }
    
}

```
## 绘制背景

``` swift

import Cocoa

class BackView: NSView {

    var image: NSImage?

    override func draw(_ dirtyRect: NSRect) {
        
        super.draw(dirtyRect)

        // Drawing code here.
        guard let image = self.image,
              let window = self.window
        else {
            return
        }
        
        let originFrame = NSRect(origin: .zero, size: window.frame.size)
        image.draw(in: originFrame, from: originFrame, operation: .sourceOver, fraction: 0.5)

    }
    
}

```