+++
title = 'Swift5：基于正则表达式实现简单的Markdown即时渲染'
date = 2021-02-13 23:23:44
+++
本文以iOS下的UITextView为例，在MacOS上只要更改NSTextView的相应的接口名同样适用。

# 实现思路

我在实现过程中尝试了局部实时渲染和全局实时渲染的方式，由于学艺不精，暂时没有找到很好的方法进行局部实时渲染，这里介绍一下我对于两种方法的思路。

## 局部实时渲染

最早的思路是根据用户的输入位置对应的段落，更新`NSMutableAttributedString`，具体步骤如下：
- 监听`textViewDidChange`
- 找到光标对应的段落
- 根据段落内容，基于正则表达式分析并对应文字的`attribute`

但是按照这种思路，在替换文字是会导致`attribute`的混乱。`attribute`的形式是`属性:Range`的形式，在更新了文本以后，所有的文本range位置都会发生变化。我暂时没有找到一个很方便的方法来局部更新`NSMutableAttributedString`的属性。

## 全局实时渲染

全局实时渲染指在监听文本变化后更新整个文本的`attribute`，这样会带来一些性能的损失，但是更易于实现。主要的思路如下：
- 监听`textViewDidChange`
- 重新解析所有文字的`attribute`并更新

我们首先需要封装一个正则表达式的工具**RE**。

# 正则表达式工具RE

RE的主要功能包括：
1. 匹配并获取`Regex`对应的内容
2. 替换`Regex`所匹配的内容
3. 匹配并获取`Regex`对应的内容及其范围(`NSRange`)

首先封装一个RE返回的结构体

```swift
struct Parsed {
    var content: String
    var range: NSRange
}
```

```swift
class RE: NSObject {
    static func regularExpression(validateString:String, inRegex regex:String) -> [String]{
        // 匹配并获取内容
        do {
            let re: NSRegularExpression = try NSRegularExpression(pattern: regex, options: [])
            let matches = re.matches(in: validateString, options:[], range: NSRange(location: 0, length: validateString.count))
            
            var data:[String] = Array()
            for item in matches {
                let string = (validateString as NSString).substring(with: item.range)
                data.append(string)
            }
            
            return data
        }
        catch {
            return []
        }
    }

    static func regularExpressionRange(validateString:String, inRegex regex:String) -> [Parsed]{
        // 匹配并获取内容和范围
        do {
            let re: NSRegularExpression = try NSRegularExpression(pattern: regex, options: [])
            let matches = re.matches(in: validateString, options:[], range: NSRange(location: 0, length: validateString.count))
            
            var data:[Parsed] = Array()
            for item in matches {
                let string = (validateString as NSString).substring(with: item.range)
                data.append(Parsed(content: string, range: item.range))
            }
            
            return data
        }
        catch {
            return []
        }
    }
    
    static func replace(validateString: String, withContent content: String, inRegex regex:String) -> String {
        do {
            let re: NSRegularExpression = try NSRegularExpression(pattern: regex, options: [])
            return re.stringByReplacingMatches(in: validateString, options: [], range: NSRange(location: 0, length: validateString.count), withTemplate: content)
        } catch {
            return validateString
        }
    }
```

# Regex

我暂时实现了以下几个markdown语法的识别：
- Header
- 带序号列表
- 无序号列表

我们需要识别出`textView`中各种类型文本及其所在的`NSRange`，对于每种类型，我们用以下几个`Regex`进行识别：

```swift
static let orderListBlockRegex = "(?<=(^|\n))([0-9]+\\..*(\n)?)*(?<=(^|\n))([0-9]+\\..*)+"
static let bulletBlocksRegex = "(?<=(^|\n))-.*(\n)?"
static let headerBlocksRegex = "(?<=(^|\n))#{1,3}.*(\n)?"
```

这三个`Regex`可以识别出多个连续的文本块(Block)何其对应的`NSRange`。

# MDParser

我封装了一个简单的Markdown解析器，返回输入的`String`经过解析后的`NSAttributedString`，具体步骤包括：
1. 为所有文本设置默认样式
2. 匹配标题并添加样式
3. 匹配有序列表，自动排序后添加样式
4. 匹配无需列表
5. 为特殊字符添加样式

```swift

class MDParser: NSObject {
    
    // 在单行文本中识别三种类型的文本
    static let headerRegex = "^#{1,3}"
    static let orderRegex = "^[0-9]+\\."
    static let bulletRegex = "^-"

    // 在多行文本中是被三种类型的文本块
    static let orderListBlockRegex = "(?<=(^|\n))([0-9]+\\..*(\n)?)*(?<=(^|\n))([0-9]+\\..*)+"
    static let bulletBlocksRegex = "(?<=(^|\n))-.*(\n)?"
    static let headerBlocksRegex = "(?<=(^|\n))#{1,3}.*(\n)?"

    // 用于给特殊字符（#，-，1.）添加样式
    static let specielRegex = "(?<=(^|\n))(#{1,3}|[0-9]+\\.|-)"
    
    // 渲染函数
    static func renderAll(content: String) -> NSAttributedString {
        let result = NSMutableAttributedString(string: content, attributes: self.normalAttribute())
        // 渲染 headers
        let headers = RE.regularExpressionRange(validateString: content, inRegex: self.headerBlocksRegex)
        for parsed in headers {
            // 获取header的等级
            let level = self.getHeaderLevel(header: parsed.content)
            // 获取header的样式
            let attributes = self.headerAttribute(level: level)
            result.addAttributes(attributes, range: parsed.range)
        }
        
        // 渲染有序列表
        let orderedList = RE.regularExpressionRange(validateString: content, inRegex: self.orderListBlockRegex)
        for parsed in orderedList {
            // 自动排序
            let replaced = RE.replace(validateString: parsed.content, withContent: "", inRegex: "(?<=(^|\n))[0-9]+")
            var splited = replaced.split(separator: "\n")
            splited = splited.enumerated().map({(i, line) in "\(i+1)"+line})
            let s = splited.joined(separator: "\n")
            result.mutableString.replaceCharacters(in: parsed.range, with: s)
            // 获取并渲染有序列表样式
            result.addAttributes(self.listAttribute(), range: parsed.range)
        }

        // 渲染无需列表
        let bulletList = RE.regularExpressionRange(validateString: content, inRegex: self.bulletBlocksRegex)
        for parsed in bulletList {
            // 获取并渲染无需列表样式
            result.addAttributes(self.listAttribute(), range: parsed.range)
        }

        // 高亮特殊字符
        for parsed in RE.regularExpressionRange(validateString: content, inRegex: self.specielRegex) {
            result.addAttributes([
                .foregroundColor: UIColor(named: "AccentColor")!
            ], range: parsed.range)
        }
        
        return result
    }
    
    // ....
    
}
```

其中，获取`header`级别以及三种样式的代码为:

```swift
// 获取header级别
static func getHeaderLevel(header: String) -> Int {
    guard let signs = RE.regularExpression(validateString: header, inRegex: "^#{1,3}").first else { return 0 }
    return signs.count
}

// MARK: - Attributes
// PARAGRAGH_LEVELS和FONT_LEVELS是自定义的全局数组，存储了各级别对应的Spacing和FontSize
private static func headerAttribute(level: Int) -> [NSAttributedString.Key: Any] {
    let paraStyle = NSMutableParagraphStyle()
    paraStyle.paragraphSpacingBefore = PARAGRAGH_LEVELS[level]
    paraStyle.paragraphSpacing = PARAGRAGH_LEVELS[level]
    let font = UIFont.boldSystemFont(ofSize: FONT_LEVELS[level])
    return [
        .font: font,
        .paragraphStyle: paraStyle,
        .foregroundColor: UIColor.label
    ]
}

private static func listAttribute() -> [NSAttributedString.Key: Any] {
    let paraStyle = NSMutableParagraphStyle()
    paraStyle.firstLineHeadIndent = 5
    paraStyle.paragraphSpacingBefore = 3
    paraStyle.paragraphSpacing = 3
    return [
        .font: UIFont.systemFont(ofSize: FONT_LEVELS[0]),
        .paragraphStyle: paraStyle,
        .foregroundColor: UIColor.label
    ]
}

private static func normalAttribute() -> [NSAttributedString.Key: Any] {
    return [
        .font: UIFont.systemFont(ofSize: FONT_LEVELS[0]),
        .foregroundColor: UIColor.label
    ]
}
```

# 实时渲染

在`ViewController`中定义`refresh()`函数，对`textView`的文本进行渲染，可以在`textViewDidChange`中调用`refresh()`函数：

```swift
final func refresh() {
    let selectedRange = self.textView.selectedRange // 保存当前光标位置
    let textStorage = self.textView.textStorage
    textStorage.setAttributedString(MDParser.renderAll(content: self.textView.text))
    self.textView.selectedRange = selectedRange // 恢复光标位置
}
```

因为在设置`AttributedString`后，光标会自动移动到最后一个字符，所以需要保存&恢复光标的位置。

``` swift
func textViewDidChange(_ textView: UITextView) {
    if self.textView.markedTextRange == nil {
        self.refresh()
    }
}
```

值得注意的是，使用中文（或是别的语言）输入法时，未键入空格也会触发`textViewDidChange`，如果此时直接刷新，则会让输入法无法使用，并直接将刚才键入的文本添加到`textView`中。

使用输入法时，`textView`会将正在输入的部分以`markedText`的形式暂存，所以判定`self.textView.markedTextRange`为空时才需要更新样式。

# 为iOS设计markdown的便捷按钮

在使用iOS端输入时，对于特殊字符（#、-或1.）的输入是十分麻烦的，所以我通过使用`textView`的`inputAccessoryView`提供了便捷的操作。

![](https://hagemon.github.io/post-images/1613312065248.PNG =200x)

便捷按钮的具体事件流程如下：
- 点击按钮
- 获取光标所在段落
- 为段落添加样式
- 刷新页面

对于不同的样式类别，也会有不同的样式更替效果：
- Header：每次点击让header的级别按照0、1、2、3循环变化
- Ordered：每次点击为段落添加或取消序号
- Bullet：每次点击为段落添加或取消无序列表符

为了实现这些功能，需要从这几个方面考虑。

## 获取段落

根据光标位置，分别向前和向后寻找段落的头尾位置（对于首段或是尾段，需要寻找文档的头尾），具体来说，使用`textView.tokenizer.position(from: toBoundary: inDirection: )`方法来寻找段落或文档的头尾。

因为`position`函数是通过寻找到上一个段落的结尾符号（也就是`\\n`)来定位段落的起始位置，所以对于第一个段落是找不到起始位置的，此时则需要找到文档的起始位置作为该段的起始位置。

```swift
final func getLineRange() -> UITextRange? {
    guard let range = self.textView.selectedTextRange
    else { return nil }
    // 以光标位置为起点
    let position = range.start
    // 向前寻找段落的边界（也就是段落起始位置）
    var start = self.textView.tokenizer.position(from: position, toBoundary: .paragraph, inDirection: UITextDirection(rawValue: UITextStorageDirection.backward.rawValue))
    // 若找不到，则将文档起点作为start
    if start == nil {
        start = self.textView.beginningOfDocument
    }
    // 向后寻找段落的边界
    var end = self.textView.tokenizer.position(from: position, toBoundary: .paragraph, inDirection: UITextDirection(rawValue: UITextStorageDirection.forward.rawValue))
    // 若找不到，则将文档终点作为end
    if end == nil {
        end = self.textView.endOfDocument
    }
    return self.textView.textRange(from: start!, to: end!)
}
```

对于三种类型的按钮，分别调用其`update`的方法更新段落：

```swift
@objc func headerAction() {
    guard let range = self.getLineRange(),
            let text = self.textView.text(in: range)
    else { return }
    self.textView.replace(range, withText: MDParser.updateHeader(s: text))
}

@objc func orderAction() {
    guard let range = self.getLineRange(),
            let text = self.textView.text(in: range)
    else { return }
    self.textView.replace(range, withText: MDParser.updateOrder(s: text))
}

@objc func bulletAction() {
    guard let range = self.getLineRange(),
            let text = self.textView.text(in: range)
    else { return }
    self.textView.replace(range, withText: MDParser.updateBullet(s: text))
}
```

值得一提的是，在`replace`之后会触发`textViewDidChange`并自动刷新，所以不需要额外的刷新操作。


## 样式更新

### Header

根据段落内容判断当前的Header等级，并让等级加一并取余：

```swift
static func updateHeader(s: String) -> String {
    if s == "\n" {return s+"# "}
    let re = RE.regularExpression(validateString: s, inRegex: self.headerRegex)
    if re.count == 0 {
        return "# " + s
    }
    let level = re[0].count
    var replaced = String(repeating: "#", count: (level + 1) % 4)
    if replaced.count > 0 {
        replaced += " "
    }
    return RE.replace(validateString: s, withContent: replaced, inRegex: self.headerRegex+"[ ]*")
}
```

### Ordered

判断段首是否为序号，若是则去除序号，否则添加序号。

```swift
static func updateOrder(s: String) -> String {
    if s == "\n" {return s+"1. "}
    let order = RE.regularExpression(validateString: s, inRegex: self.orderRegex)
    if order.count > 0 {
        return RE.replace(validateString: s, withContent: "", inRegex: self.orderRegex+"[ ]*")
    }
    // This would be auto reordered, using `1.` works fine.
    return "1. "+s
}
```

值得注意的是，由于在之后会自动刷新并重新排序，所以这里添加任意的序号都可以。

### Bullet

和Ordered原理相同

```swift
static func updateBullet(s: String) -> String {
    if s == "\n" {return s+"- "}
    let bullet = RE.regularExpression(validateString: s, inRegex: self.bulletRegex)
    if bullet.count > 0 {
        return RE.replace(validateString: s, withContent: "", inRegex: self.bulletRegex+"[ ]*")
    }
    return "- "+s
}
```