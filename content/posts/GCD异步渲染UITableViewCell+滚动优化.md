+++
title = 'GCD异步渲染UITableViewCell+滚动优化'
date = 2021-02-25 22:16:05
+++

在基于feed流的应用中，有两个需要重点解决的问题：

- 异步进行耗时的数据加载
- 滚动时控制加载任务以保持滚动流畅

设定一个简单的场景：

- 基于UITableViewCell
- 每个cell都拥有一个UIImageView
- 在UIImageView上绘制圆角头像
- 绘制100个cell并进行高频率的滚动

我们可以使用Instrument和XCode的Debugger里观察这些指标：

- Average Frame Time
- 是否离屏渲染

# 简单实现

最简单的实现方式是为每一个cell上的UIImageView设置cornerRadius，但是会存在离屏渲染，且AFT很长。
其实现在直接用png并设置cornerRadius并不会被离屏渲染，这里为了展示效果特地把layer栅格化一下😅

``` swift
class TableViewController: UITableViewController {
    
    override func viewDidLoad() {
        super.viewDidLoad()
    }

    override func numberOfSections(in tableView: UITableView) -> Int {
        return 1
    }

    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return 100
    }

    
    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
        imageView.image = UIImage(named: "\(indexPath.row % 8 + 1).png")
        imageView.clipsToBounds = true
        imageView.layer.cornerRadius = imageView.bounds.width / 2
        imageView.layer.shouldRasterize = true
        return cell
    }
    
    override func tableView(_ tableView: UITableView, heightForRowAt indexPath: IndexPath) -> CGFloat {
        return 50
    }

}
```

检测屏幕上离屏渲染的部分：

![](https://hagemon.github.io/post-images/1614265823958.PNG =300x)

因为在iPhone12 Pro上调试，所以在Instrument里的帧数还是挺高的😅

# 优化

接下来将从：

- 异步渲染
- 绘制圆角
- 滚动优化

这几个方面进行优化

## 绘制圆角

为UIImage写一个extension，基于UIBezierPath和Context绘制圆角图像：

``` swift
extension UIImage {
    func roundedImage(backgroundColor: UIColor=UIColor.white) -> UIImage? {
        let rect = CGRect(origin: CGPoint.zero, size: self.size)
        let size = self.size
        UIGraphicsBeginImageContextWithOptions(size, true, 0)
        backgroundColor.setFill()
        UIRectFill(rect)
        let path = UIBezierPath(ovalIn: rect)
        path.addClip()
        self.draw(in: rect)
        let rounded = UIGraphicsGetImageFromCurrentImageContext()
        UIGraphicsEndImageContext()
        return rounded
    }
}
```

## 异步渲染

绘制圆角矩形是一个相对耗时的操作，可以把它放在global线程里进行，在绘制完成后异步调用主线程进行UI更新。

``` swift
override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
    let imageView = cell.viewWithTag(1) as! UIImageView

    DispatchQueue.global().async {
        let image = UIImage(named: "\(indexPath.row % 8 + 1).png")!
        let rounded = image.roundedImage()
        DispatchQueue.main.async {
            imageView.image = rounded
        }
    }
    return cell
}
```

## 滚动优化

只是用异步渲染的话，在高频率滚动的情况下可能会对出现过的所有cell都进行加载，很容易出现内存溢出的情况，所以需要控制这些加载任务的执行。

具体的实现思路为：

- 使用一个字典来保存当前需要渲染的任务
    - key为indexPath.row
    - value为需要渲染的任务
- 使用DispatchWorkItem来封装加载任务
- 在cellForRow函数中异步执行加载任务并添加到字典中
- 在cellDidEndDisplaying函数中取消不限时的cell的任务，并移除出字典

``` swift
import UIKit

class TableViewController: UITableViewController {
    
    // 用于保存图像渲染任务
    var items: [String: DispatchWorkItem] = [:]

    override func viewDidLoad() {
        super.viewDidLoad()
    }

    // MARK: - Table view data source

    override func numberOfSections(in tableView: UITableView) -> Int {
        // #warning Incomplete implementation, return the number of sections
        return 1
    }

    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return 100
    }

    
    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
        let imageView = cell.viewWithTag(1) as! UIImageView

        // 封装渲染任务
        let item = DispatchWorkItem {
            let image = UIImage(named: "\(indexPath.row % 8 + 1).png")!
            let rounded = image.roundedImage()
            DispatchQueue.main.async {
                // 主线程异步更新UI
                imageView.image = rounded
            }
        }
        // 添加到字典中
        self.items.updateValue(item, forKey: "\(indexPath.row)")
        // global线程异步执行渲染任务
        DispatchQueue.global().async {
            item.perform()
        }
        return cell
    }
    
    override func tableView(_ tableView: UITableView, heightForRowAt indexPath: IndexPath) -> CGFloat {
        return 50
    }
    
    override func tableView(_ tableView: UITableView, didEndDisplaying cell: UITableViewCell, forRowAt indexPath: IndexPath) {
        guard let item = self.items["\(indexPath.row)"] else {return}
        // 取消已经被移除的cell中的任务
        item.cancel()
        // 从字典中移除对应任务
        self.items.removeValue(forKey: "\(indexPath.row)")
    }

}
```

可以看到，已经没有了离屏渲染的现象：

![](https://hagemon.github.io/post-images/1614267273643.PNG =300x)

在Instrument中滚动也保持着60的FPS。

