+++
title = 'Clipin Log 4. 高亮区域的选取&拖拽&调整'
date = 2021-01-09 21:25:34
tags = ['iOS dev']
+++

# 高亮区域的选取

确定了高亮区域的渲染方式后，我们在ClipWindowController中添加对鼠标事件的监听，用于获取高亮区域。

在ClipWindowController中设置一些变量，用来记录用户操作过程中的一些NSPoint和NSRect：

```swift
class ClipWindowController: NSWindowController {
    
    
    var clipView: ClipView?
    var screenImage: NSImage?
    
    var lastRect: NSRect?        // 上一个完整的区域，用于保存拖拽或调整前的区域
    var highlightRect: NSRect?   // 当前高亮的区域，通过鼠标操作动态更新
    var startPoint: NSPoint?     // 上一个完整区域的origin
    var lastPoint: NSPoint?      // 上一次更新区域时鼠标的位置
    
    // functions ....
}
```

## 实现思路：

- 鼠标落下时状态变为start，并记录落下位置为高亮区域的startPoint
- 鼠标拖拽时通过当前位置和startPoint创建新的Rect，用于更新highlightRect

其中用到`mouseDown`、`mouseUp`、`mouseDrag`这三个事件监听：

```swift
override func mouseDown(with event: NSEvent) {
      let location = event.locationInWindow
      switch ClipManager.shared.status {
      case .ready:
          ClipManager.shared.status = .start
          self.startPoint = location
      case .select:
          // 暂不实现
          break
      default:
          return
      }
  }
```

```swift
override func mouseUp(with event: NSEvent) {
    switch ClipManager.shared.status {
    case .start:
        guard let rect = self.highlightRect, let view = self.clipView else {
            // 没有形成高亮区域
            ClipManager.shared.status = .ready
            return
        }
        ClipManager.shared.status = .select
        self.lastRect = rect
        self.highlight()
    case .drag, .adjust:
        // 暂不实现
        break
    default:
        return
    }
}
```

```swift
override func mouseDragged(with event: NSEvent) {
    let location = event.locationInWindow
    switch ClipManager.shared.status {
    case .start:
        guard let start = self.startPoint else { return }
        // 根据 startPoint和当前位置构建新的rect并更新highlightRect
        self.highlightRect = RectUtil.getRect(aPoint: start, bPoint: location)
        // 重新渲染
        self.highlight()
    case .adjust:
        // 暂不实现
        break
    case .drag:
         // 暂不实现
        break
    default:
        break
    }
}
```

其中`RectUtil.getRect`是在工具类中的一个静态方法，根据两个Point获取对应的Rect：

Utils/RectUtil.swift

```swift
class RectUtil: NSObject {
	  static func getRect(aPoint: NSPoint, bPoint:NSPoint) -> NSRect{
	      return NSIntegralRect(NSRect(
	          x: min(aPoint.x, bPoint.x),
	          y: min(aPoint.y, bPoint.y),
	          width: abs(aPoint.x-bPoint.x),
	          height: abs(aPoint.y-bPoint.y)
	      ))
	  }
	  // ... other functions
}
```

# 高亮区域的拖拽

## 实现思路：

- 判断mouseDown是否在高亮区域内部
- mouseUp时保存当前高亮区域信息
- mouseDragged时计算偏移程度，同时检测rect是否超出屏幕，更新highlightRect

```swift
override func mouseDown(with event: NSEvent) {
      let location = event.locationInWindow
      switch ClipManager.shared.status {
      case .ready:
          // ....
      case .select:
          // 判断鼠标区域
          guard let rect = self.highlightRect,
                    rect.contains(location)
          else { return }
          ClipManager.shared.status = .drag
          self.lastPoint = location
      default:
          return
      }
  }
```

```swift
override func mouseUp(with event: NSEvent) {
    switch ClipManager.shared.status {
    case .start:
        // ....
    case .drag, .adjust:
        // 保存高亮区域信息
        guard let rect = self.highlightRect else { return }
        self.startPoint = rect.origin
        self.selectDotType = .none
        self.lastRect = rect
        ClipManager.shared.status = .select
    default:
        return
    }
}
```

```swift
override func mouseDragged(with event: NSEvent) {
    let location = event.locationInWindow
    switch ClipManager.shared.status {
        case .start:
            // ....
        case .adjust:
            // 暂不实现
            break
        case .drag:
            guard var rect = self.highlightRect,
                    let window = self.window
            else { break }
            
            // 和上一次检测的鼠标位置的偏移
            var dx = location.x - self.lastPoint!.x
            var dy = location.y - self.lastPoint!.y

            // 偏移后的rect
            let offsetRect = rect.offsetBy(dx: dx, dy: dy)
            switch RectUtil.detectOverflow(rect: offsetRect, in: window) {
            case .bothOverflow:
                // 如果x和y都溢出，则抹零两轴的偏移量
                dx = 0
                dy = 0
            case .xOverflow:
                // x溢出，抹零x轴偏移量
                dx = 0
            case .yOverflow:
                // y溢出，抹零y轴偏移量
                dy = 0
            default:
                break
            }
            rect = rect.offsetBy(dx: dx, dy: dy)
        
            // 更新highlightRect，记录鼠标位置
            self.highlightRect = rect
            self.lastPoint = location
            self.highlight()
        default:
            break
    }
}
```

其中rect溢出检测实现为：

```swift
class RectUtil: NSObject {
		static func detectOverflow(rect: NSRect, in window:NSWindow) -> RectIssue {
        let origin = rect.origin
        let points = [origin,
                      origin.offsetBy(dx: rect.width, dy: 0),
                      origin.offsetBy(dx: 0, dy: rect.height),
                      origin.offsetBy(dx: rect.width, dy: rect.height)
        ]
        var xFlag = false
        var yFlag = false
        for p in points {
            if p.x < 0 || p.x > window.frame.width{
                xFlag = true
            }
            if p.y < 0 || p.y > window.frame.height{
                yFlag = true
            }
        }
        if xFlag && yFlag {
            return .bothOverflow
        }else if xFlag {
            return .xOverflow
        }else if yFlag {
            return .yOverflow
        }else{
            return .normal
        }
    }
    // ... other functions
}

enum RectIssue {
    case xOverflow
    case yOverflow
    case bothOverflow
    case normal
}
```

# 高亮区域的调整

## 实现思路：

- 为高亮区域添加八个标识点
- 若mouseDown在这八个点内则进入adjust状态
- mouseUp时保存高亮区域信息（和拖拽相同）
- mouseDragged时判断点击了哪个标识点
    - 若为四个角落的点(`.corner`)则找到该点在`lastRect`上中心对称的点作为**定点**，鼠标位置作为**动点**；
    - 若是四条边上的点，则将其**对边的其中一个端点**作为**定点**，将当前鼠标位置所在边上与定点对称的点作为**动点**，如图所示：

![](https://hagemon.github.io/post-images/1613223037730.jpg)

## 标识点显示

### 标识点类型：

```swift
enum DotType {
    case corner  // 四个角的点
    case top     // 上边中心对应的点
    case bottom  // 下边中心对应的点
    case right   // 右边中心对应的点
    case left    // 左边中心对应的点
    case none    // 无类型标识点，当没有标识点被选到时的默认值
}
```

### 实现思路：

- 在view中根据当前的drawingRect构造八个点的NSPoint
- 使用NSBezierPath绘制八个标识点

```swift
class ClipView: NSView {
    
    var image: NSImage?
    var drawingRect: NSRect?
    var showDots = false  // 是否显示标识点
    var paths: [(NSBezierPath, DotType)] = [] // 记录标识点的path和对应的类型

    override func draw(_ dirtyRect: NSRect) {
        
        super.draw(dirtyRect)

        // Drawing code here.
        guard let image = self.image else {
            return
        }
        var rect = NSIntersectionRect(self.drawingRect!, self.bounds)
        rect = NSIntegralRect(rect)
        image.draw(in: rect, from: rect, operation: .sourceOver, fraction: 1.0)
				
				// 根据当前drawingRect绘制标识点
        if self.showDots {
            self.drawDots(rect: rect)
        }

    }
    
    private func drawDots(rect: NSRect) {
        let radius: CGFloat = 1.5 // 标识点半径
        let dots = self.getDotsCoord(with: rect, radius: radius)  // 获取八个点的坐标
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
				// 为了让标识点的中心落在正确的位置，根据半径提前偏移rect
				// 计算出来的位置是标识点圆圈对应矩阵的左下角（起始坐标）
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
						// 取整
            return (NSPoint(x: Int(point.x), y: Int(point.y)), type)
        })
        return dots
    }   
    
}
```

## 区域调整

在Controller中记录当前选择的标识点，并在mouseDown中添加对adjust or drag的判断

```swift
class ClipWindowController: NSWindowController {
    
    
    var clipView: ClipView?
    var screenImage: NSImage?
    
    var lastRect: NSRect?  
    var highlightRect: NSRect?  
    var startPoint: NSPoint?     
    var lastPoint: NSPoint?    

    var selectDotType: DotType = .none  // 当前选择标识点的类型，默认为none，可以用于判断是否有选择标识点
    var selectDot: NSPoint = .zero      // 当前选择的标识点，默认为zero
    
		// other functions ....

    override func mouseDown(with event: NSEvent) {
        let location = event.locationInWindow
        switch ClipManager.shared.status {
        case .ready:
            // ...
        case .select:
            guard let rect = self.highlightRect,
                     let view = self.clipView,
                     rect.insetBy(dx: -5, dy: -5).contains(location) // 适当放宽点击判定区域
            else { return }
            for (path, type) in view.paths {
								 // 适当放宽点击判定区域
                if path.bounds.insetBy(dx: -5, dy: -5).contains(location) {
                    self.selectDotType = type
                    self.selectDot = path.bounds.center()
                    break
                }
            }
            if self.selectDotType != .none {
                ClipManager.shared.status = .adjust
            } else {
                ClipManager.shared.status = .drag
            }
            self.lastPoint = location
        default:
            return
        }
    }

    override func mouseUp(with event: NSEvent) {
        switch ClipManager.shared.status {
        case .start:
            // ...
        case .drag, .adjust:
            guard let rect = self.highlightRect else { return }
            self.startPoint = rect.origin
            self.selectDotType = .none
            self.lastRect = rect
            ClipManager.shared.status = .select
        default:
            return
        }
    }

    override func mouseDragged(with event: NSEvent) {
        let location = event.locationInWindow
        switch ClipManager.shared.status {
        case .start:
            // ....
        case .adjust:
            guard let lastRect = self.lastRect,
                  self.selectDotType != .none
            else { break }
            let dx = location.x - self.selectDot.x
            let dy = location.y - self.selectDot.y
            var rect: NSRect = .zero
            switch self.selectDotType {
            case .corner:
                let symPoint = lastRect.symmetricalPoint(point: self.selectDot)
                rect = RectUtil.getRect(aPoint: symPoint, bPoint: location)
            case .top:
                rect = RectUtil.getRect(
                    aPoint: lastRect.origin,
                    bPoint: self.selectDot.offsetBy(dx: lastRect.width/2, dy: dy)
                )
            case .bottom:
                rect = RectUtil.getRect(
                    aPoint: lastRect.origin.offsetBy(dx: 0, dy: lastRect.height),
                    bPoint: self.selectDot.offsetBy(dx: lastRect.width/2, dy: dy)
                )
            case .left:
                rect = RectUtil.getRect(
                    aPoint: lastRect.origin.offsetBy(dx: dx, dy: 0),
                    bPoint: self.selectDot.offsetBy(dx: lastRect.width, dy: lastRect.height/2)
                )
            case .right:
                rect = RectUtil.getRect(
                    aPoint: lastRect.origin,
                    bPoint: self.selectDot.offsetBy(dx: dx, dy: lastRect.height/2)
                )
            
            default:
                break
            }
            self.highlightRect = rect
            self.lastPoint = location
            self.startPoint = rect.origin
            self.highlight()
        case .drag:
            // ...
        default:
            break
        }
    }
}
```

其中`symmetricalPoint`是获取rect中某个点关于中心对称的点：

```swift
extension NSRect {
    func center() -> NSPoint{
        let origin = self.origin
        return NSPoint(x: origin.x+self.width/2, y: origin.y+self.height/2)
        
    }
    
    func symmetricalPoint(point: NSPoint) -> NSPoint {
        let center = self.center()
				// 转换为整数，放置抖动
        return NSPoint(x: Int(2*center.x-point.x), y: Int(2*center.y-point.y))
    }
}
```

接下来是[单屏幕Pin实现](https://hagemon.github.io/post/clipin-log-5)