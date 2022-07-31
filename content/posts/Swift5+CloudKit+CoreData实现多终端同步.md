---
title: 'Swift5+CloudKit+CoreData实现多终端同步'
date: 2021-02-04 22:42:18
tags: [iOS]
published: true
hideInList: false
feature: 
isTop: false
---
CloudKit和CoreData的配置已经有很多资料了，这里不阐述。

# Preparation

在基于CoreData和CloudKit的项目中，会在AppDelegate中自动生成container：

```swift
lazy var persistentContainer: NSPersistentContainer = {
    
    let container = NSPersistentContainer(name: "Model")
    
    // get the default store description
    guard let description = container.persistentStoreDescriptions.first else {
        fatalError("Could not retrieve a persistent store description.")
    }

    // initialize the CloudKit schema
    let id = "iCloud.your.id"
            // 获取基于CloudKit的Options
    let options = NSPersistentCloudKitContainerOptions(containerIdentifier: id)
    description.cloudKitContainerOptions = options
            // 用于跟踪merge的记录
            description.setOption(true as NSNumber, forKey: NSPersistentHistoryTrackingKey)
    container.persistentStoreDescriptions = [description]
    container.loadPersistentStores(completionHandler: {
        (storeDescription, error) in
        if let error = error as NSError? {
            fatalError("Unresolved error \(error), \(error.userInfo)")
        }
    })
    return container
}()
```

我使用了一个CoreUtil类封装了以下属性和方法：

- container
- context
- lastToken
- 数据库操作

```swift
final class CoreUtil: NSObject {
    
    // MARK: - Core Data Context
    static let container = (UIApplication.shared.delegate as! AppDelegate).persistentContainer
    static let context = container.viewContext

    // 暂时不展开token
    static var lastToken: NSPersistentHistoryToken? = ...
    static var tokenFile: URL = ...

    // ..数据库操作
    // ...
}
```

## Model

Model中包含了三个属性：

- title
- content
- date

# 终端→云端

当拥有了基于CloudKit的context时，所有的数据库操作在save之后都会自动同步到iCloud，所以只需要实现基本的CURD操作，就可以做到将数据同步到云端。

```swift
final class CoreUtil: NSObject {
    
    // MARK: - Core Data Context
    // ...
    
    // MARK: - Database Operations

    // Creation
    static func create(title: String, content: String, date: Date) -> CoreMemo {
        let entity = NSEntityDescription.entity(forEntityName: "Model", in: self.context)
        let model = NSManagedObject(entity: entity!, insertInto: self.context) as! Model
        model.setValue(title, forKey: "title")
        model.setValue(content, forKey: "content")
        model.setValue(date.toString(), forKey: "date")
        return model
    }

    // Removal

    static func remove(model: Model) {
        do {
            self.context.delete(model)
            try self.context.save()
        } catch {
            print("Delete \(model.title ?? "nil") failed")
            print(error)
        }
    }

    // Retrieve

    static func getModels() -> [Model] {
        let request = NSFetchRequest<NSFetchRequestResult>(entityName: "Model")
        // 按时间排序
        request.sortDescriptors = [NSSortDescriptor(key: "date", ascending: false)]
        request.returnsObjectsAsFaults = false
        do {
            let result = try context.fetch(request) as! [Model]
            // 尝试输出
            for data in result {
               print(data.value(forKey: "title") as! String, data.value(forKey: "date") as! String)
            }
            return result
        } catch {
            print("Load Failed")
            return []
        }
    }
    
    // Saving & Update

    static func save() {
        do {
            try self.context.save()
        } catch {
            print("Store failed")
        }
    }
    
}
```

上述的数据库操作中没有修改(Update)操作，因为在修改对象的值之后，只需要保存当前的context既可以完成对数据库中数据的修改。

我使用了Model的extension完成Model对象的修改操作（因为我使用了CoreData自动生成的Model类，所以看不到Model类的源代码）：

```swift
extension Model {
    func update(content: String){
        self.setValue(content, forKey: "content")
        // Parser.getTitle是我根据content获取标题的方法，大家可以自由发挥
        self.setValue(Parser.getTitle(content: content), forKey: "title")
        self.setValue(Date.now().toString(), forKey: "date")
    }
}
```

在App中对Model对象执行数据库操作后，数据会自动同步到iCloud中。

# 云端→终端

我们在启动App以后，container会连接到iCloud并获取最新的数据。当云端数据变化时，App会收到从云端发来的变化通知，但在默认情况下并不会将数据合并（merge）到当前的context之中。

我们使用iCloud的一个重要的需求就是实时地从iCloud获取其它设备对iCloud的修改，所以我们需要监听iCloud的行为，当其发送远端数据变化的通知时，执行相应的更新操作。

## AppDelegate

为了监听远端的通知，需要为container的description添加新的参数：

```swift
description.setOption(true as NSNumber, forKey: NSPersistentHistoryTrackingKey)
// 新的参数：表示是否接受远端变化的通知
description.setOption(true as NSNumber, forKey: NSPersistentStoreRemoteChangeNotificationPostOptionKey)
```

## ViewController

在ViewDidLoad中添加对事件的监听：

```swift
override func viewDidLoad() {
    super.viewDidLoad()
		NotificationCenter.default.addObserver(self,
                                           selector: #selector(self.handleRemoteData(_:)),
                                           name: .NSPersistentStoreRemoteChange,
                                           object: CoreUtil.container.persistentStoreCoordinator)
}
```

实现对通知的处理：

```swift
@objc func handleRemoteData(_ notification: Notification) {
    guard let info = notification.userInfo
    else {
        return
    }
    // ... 一些处理
}
```

在默认情况下，当App接收到新的变化时，是不会merge到当前的context之中的，但是会保留一个history，作为云端数据变化的记录。为了让源端数据合并，我们可以使用以下两个方法：

## 方法一：为context设置自动合并（不推荐）

一个比较简单的实现方法是为context设置自动合并，但是官方并不推荐这种做法：

```swift
container.persistentStoreDescriptions = [description]
container.loadPersistentStores(completionHandler: {
    (storeDescription, error) in
    if let error = error as NSError? {
        fatalError("Unresolved error \(error), \(error.userInfo)")
    }
})
// 不推荐的做法
container.viewContext.automaticallyMergesChangesFromParent = true
return container
```

当App收到远端的变化时，会自动的将变化merge到context之中，值得注意的是，这个merge的过程是在**子线程异步执行的**。所以在我们的handle函数执行的时候，这个merge的行为**并不一定完成**。

```swift
@objc func handleRemoteData(_ notification: Notification) {
    guard let info = notification.userInfo
    else {
        return
    }
		DispatchQueue.main.async {
        [unowned self] in
        // 此时merge可能还未完成
        self.tableView.reloadData()
    }
}
```

另外，这个handle函数也会在子线程进行，所以对tableView的刷新要在主线程中异步进行。

值得注意的是，因为一些机制（我还未探究原因），远端在**一次**数据变化后，会发送**多个**通知到App，这个handle函数也会执行多次，如果按照这样实现的话其实是可以达到刷新出新数据的效果的（在第2次执行这个函数的时候，之前的merge可能已经完成了），但多次的刷新势必会带来一定的性能损耗。

一个想法是在CoreUtil中设置一个lastToken，根据userInfo里的token信息来判断是否已经获取过本次数据变化：

```swift
@objc func handleRemoteData(_ notification: Notification) {
    guard let info = notification.userInfo,
          let token = info["historyToken"] as? NSPersistentHistoryToken,
          token != CoreUtil.lastToken
    else {
        return
    }
    guard let trans = CoreUtil.getLatestTransaction(token: token) else {return}
    DispatchQueue.main.async {
        [unowned self] in
        // 此时merge可能还未完成
        self.tableView.reloadData()
        CoreUtil.lastToken = token
    }
}
```

但是面临着一个同样的问题，我们并不能知道在哪一次的通知中数据才被merge到context之中，所以草率地在reloadData()后修改lastToken，并不会让tableView得到最新的数据。（这也是我踩过的坑）

## 手动Merge远端变化

按照官方文档的建议，还是需要手动merge远端的变化，然后再执行相应的UI操作。

在CoreUtil中，我们将lastToken保存在文件中，每次读取的时候从文件中查看当前最新的token信息：

```swift
final class CoreUtil: NSObject {
    
    // MARK: - Core Data Context
    static let container = (UIApplication.shared.delegate as! AppDelegate).persistentContainer
    static let context = container.viewContext

    // 暂时不展开token
    static var lastToken: NSPersistentHistoryToken? = nil {
        didSet{
            guard let token = lastToken,
                  let data = try? NSKeyedArchiver.archivedData(withRootObject: token, requiringSecureCoding: true)
            else { return }
            do {
                try data.write(to: tokenFile)
            } catch {
                fatalError(error.localizedDescription)
            }
        }
    }
    static var tokenFile: URL = {
        let url = NSPersistentContainer.defaultDirectoryURL().appendingPathComponent("TapMemo", isDirectory: true)
        if !FileManager.default.fileExists(atPath: url.path) {
            do {
                try FileManager.default.createDirectory(at: url, withIntermediateDirectories: true, attributes: nil)
            } catch {
                fatalError(error.localizedDescription)
            }
        }
        return url.appendingPathComponent("token.data", isDirectory: false)
    }()

    // ..数据库操作
    // ...
}
```

在监听通知时，我们需要做以下几件事情：

1. 通过判断当前token和最新的token是否一致，来确定是否已经被更新过
2. 获取最新的historyRequest
3. 确保获取了这个request以后，将最新的token设置为当前收到的token
4. 获取该history中最新事务（transaction）信息
5. merge该事务的变化
6. 在主线程中更新UI

```swift
@objc func handleRemoteData(_ notification:Notification) {
    guard let info = notification.userInfo,
          let token = info["historyToken"] as? NSPersistentHistoryToken,
          token != CoreUtil.lastToken
    else {
        return
    }
    let fetchHistoryRequest = NSPersistentHistoryChangeRequest.fetchHistory(after: self.token)
    CoreUtil.lastToken = token
    let context = CoreUtil.context
    guard
        let historyResult = try? context.execute(fetchHistoryRequest)
            as? NSPersistentHistoryResult,
        let history = historyResult.result as? [NSPersistentHistoryTransaction]
    else {
        print("Could not convert history result to transactions.")
        return
    }
    guard history.count > 0,
          let trans = history.last
    else {
        return
    }
    CoreUtil.context.perform {
        CoreUtil.context.mergeChanges(fromContextDidSave: trans.objectIDNotification())
        DispatchQueue.main.async {
            [unowned self] in
            self.tableView.reloadData()
        }
    }
}
```

这样我们就可以在多个设备中同步信息啦！