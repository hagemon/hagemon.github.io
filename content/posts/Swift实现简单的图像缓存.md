+++
title = 'Swift实现简单的图像缓存'
date = 2021-02-28 22:07:38
tags = ['iOS dev']
+++
在iOS App中，为了增加用户的体验感、减少网络的请求、增加App的性能，我们在获取图像之后通常会将其进行缓存，我学习了[从零开始打造一个iOS图片加载框架](https://juejin.cn/post/6844903807667666951)，写了一个简单的基于Swift的图像缓存。

## 图片加载

我们从最简单的场景开始，在ViewController中实现download方法，通过URLSession下载一个图片并显示到ImageView上：

```swift
func download() {
    let url = URL(string: "https://user-gold-cdn.xitu.io/2019/3/25/169b406dfc5fe46e")
    let session = URLSession.shared
    let task = session.dataTask(with: url!, completionHandler: { [weak self]
        data, res, error in
        guard error == nil,
              let data = data
        else { return }
        let image = UIImage(data: data)
        guard let strongSelf = self else {return}
        DispatchQueue.main.async {
            strongSelf.imageView.image = image
        }
    })
    task.resume()
}
```

## 图片缓存

上述的方法在每次读取页面之后都会进行下载。为了达到缓存的目的，我们创建一个工具类，以单例模式来封装图片的下载和缓存的功能。使用NSCache对图像进行缓存，当下载了以后，若后续再次需要读取这个url，则先从缓存中寻找。

```swift
class ImageDownloader: NSObject {
    static let shared = ImageDownloader()
    let session: URLSession
    let cache: NSCache<NSString, UIImage>
    
    override init() {
        self.session = URLSession.shared
        self.cache = NSCache<NSString, UIImage>()
    }
    
    func download(urlString: NSString, completionHandler: @escaping (UIImage?, NSError?) -> ()) {
        if urlString.length == 0 {
            return
        }
        let cached = self.cache.object(forKey: urlString)
        if cached != nil {
            print("load \(urlString) from cache")
            completionHandler(cached, nil)
            return
        }
        guard let url = URL(string: urlString as String) else { return }
        let task = self.session.dataTask(with: url, completionHandler: {
            [weak self] data, res, error in
            guard error == nil,
                  let data = data,
                  let strongSelf = self,
                  let image = UIImage(data: data)
            else {return}
            print("\(urlString) cached")
            strongSelf.cache.setObject(image, forKey: urlString)
            DispatchQueue.main.async {
                completionHandler(image, nil)
            }
        })
        task.resume()
    }
}
```

## 磁盘缓存

使用NSCache仅能把图像缓存在内存中，在下次开启app之后又要重新下载，所以需要把Data保存在磁盘之中。我们进一步把缓存功能从下载器中提取出来，封装到缓存器中。

在初始化缓存器的时候，需要进行：

- 内存cache创建
- fileManager的创建
- 创建数据在disk保存的路径

```swift
override init() {
    self.cache = NSCache<NSString, UIImage>()
    self.fileManager = FileManager()
    do {
        self.path = try self.fileManager.url(for: .cachesDirectory, in: .userDomainMask, appropriateFor: nil, create: true).appendingPathComponent("hagemon.Cache")
        if !self.fileManager.fileExists(atPath: self.path.absoluteString) {
            try self.fileManager.createDirectory(at: self.path, withIntermediateDirectories: true, attributes: nil)
        }
    } catch {
        fatalError("Failed to create cache on disk")
    }
}
```

同时还要实现将url的哈希，和获取文件路径的方法：

```swift
private func keyForURL(urlString: String) -> String {
    return "\(urlString.hash)"
}

private func urlForImage(withUrlString urlString: String) -> URL {
    let key = self.keyForURL(urlString: urlString)
    let url = self.path.appendingPathComponent(key)
    return url
}
```

在获取图像缓存的时候，首先需要判断是否在内存中缓存，再检查是否在磁盘中有缓存：

```swift
func fetchCachedImage(with urlString: String) -> UIImage? {
    guard urlString.count > 0 else {return nil}
	// check memory cache
    if let image = self.cache.object(forKey: urlString as NSString) {
        print("load \(urlString) from memory cache")
        return image
    }
    let url = self.urlForImage(withUrlString: urlString)
    do {
		// load data from url
        let data = try Data(contentsOf: url)
        guard let image = UIImage(data: data) else { return nil }
        self.cache.setObject(image, forKey: urlString as NSString)
        print("load \(urlString) from \(url.absoluteString)")
        return image
    } catch {
		// url not exists
        return nil
    }
}
```

在保存图像的时候，将图像保存到内存cache中，并在编码后保存到磁盘中：

```swift
func storeImage(image: UIImage, forKey key: String) {
	// store to memory cache
    self.cache.setObject(image, forKey: key as NSString)
    print("store \(key) to memory cache")
    guard let data = image.pngData() else {return}
    do {
        let url = self.urlForImage(withUrlString: key)
		// store to disk
        try data.write(to: url)
        print("store \(key) to \(url.absoluteString)")
    } catch {
        print(error)
        fatalError("failed to cache \(key)")
    }
    
}
```

在Downloader中，就可以把缓存的使用改编为：

```swift
class ImageDownloader: NSObject {
    static let shared = ImageDownloader()
    let session: URLSession
    
    override init() {
        self.session = URLSession.shared
    }
    
    func download(urlString: String, completionHandler: @escaping (UIImage?, NSError?) -> ()) {
        if urlString.count == 0 {
            return
        }
        let cached = ImageCache.shared.fetchCachedImage(with: urlString)
        if cached != nil {
            completionHandler(cached, nil)
            return
        }
        guard let url = URL(string: urlString) else { return }
        let task = self.session.dataTask(with: url, completionHandler: {
            data, res, error in
            guard error == nil,
                  let data = data,
                  let image = UIImage(data: data)
            else {return}
            ImageCache.shared.storeImage(image: image, forKey: urlString)
            DispatchQueue.main.async {
                completionHandler(image, nil)
            }
        })
        task.resume()
    }
}
```

## 异步缓存

有时候图像在磁盘上的存储和读写是十分消耗时间的，我们可以把这个过程放在后台进行，当存取完毕之后再更新UI。

于是我们可以将获取缓存的方法由返回数据的形式改为回调的形式：

```swift
func fetchCachedImage(with urlString: String, completionHandler: @escaping (_ image: UIImage?) -> ()){
    guard urlString.count > 0 else {
        completionHandler(nil)
        return
    }
    if let image = self.cache.object(forKey: urlString as NSString) {
        print("load \(urlString) from memory cache")
        completionHandler(image)
    }
    let url = self.urlForImage(withUrlString: urlString)
    do {
        let data = try Data(contentsOf: url)
        guard let image = UIImage(data: data) else {
            completionHandler(nil)
            return
        }
        self.cache.setObject(image, forKey: urlString as NSString)
        print("load \(urlString) from \(url.absoluteString)")
        completionHandler(image)
    } catch {
        completionHandler(nil)
    }
}
```

在Downloader的调用中可以改为：

```swift
func download(urlString: String, completionHandler: @escaping (UIImage?, NSError?) -> ()) {
    if urlString.count == 0 {
        return
    }
    ImageCache.shared.fetchCachedImage(with: urlString) {
        cached in
        if cached != nil {
            completionHandler(cached, nil)
            return
        }
        guard let url = URL(string: urlString) else { return }
        let task = self.session.dataTask(with: url, completionHandler: {
            data, res, error in
            guard error == nil,
                  let data = data,
                  let image = UIImage(data: data)
            else {return}
            ImageCache.shared.storeImage(image: image, forKey: urlString)
            DispatchQueue.main.async {
                completionHandler(image, nil)
            }
        })
        task.resume()
    }
}
```