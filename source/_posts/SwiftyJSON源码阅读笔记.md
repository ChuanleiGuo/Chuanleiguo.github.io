---
layout: post
title:  SwiftyJSON 源码阅读笔记
date:   2016-7-10 10:18:23 +0800
categories: 
- 源码分析
tags:
- swift
- sourcecode
---

SwiftyJSON 是使用Swift编写的 JSON 解析库。Swift 是一门强类型的语言，并且引入了 Optional 类型。这使得解析 JSON 数据变得异常麻烦，我们往往需要不断地判断 Optional 中是否包含了有效值：

``` swift
let jsonObject : AnyObject! = NSJSONSerialization.JSONObjectWithData(dataFromTwitter, options: NSJSONReadingOptions.MutableContainers, error: nil)
if let statusesArray = jsonObject as? NSArray{
    if let aStatus = statusesArray[0] as? NSDictionary{
        if let user = aStatus["user"] as? NSDictionary{
            if let userName = user["name"] as? NSDictionary{
                
            }
        }
    }
}
```

使用可选链的来访问 JSON 中的数据也依然非常麻烦。

SwiftyJSON 很好的解决了上面的问题，使我们可以用简洁的语法访问数据，而不用担心程序崩溃。正是因为安全、高效以及极易上手的特点，它在Github获得了超过10000 Star。

“优秀的源代码使最好的老师”。我们来研究一下 SwiftyJSON 是怎样实现的。

## 数据保存

SwiftJSON 使用一个 Struct 来抽象表示 JSON：

``` swift
public struct JSON {

}
```

我们在[json.org](http://www.json.org)看到，JSON 只可以存储几种简单的数据：

1. 键值对，即字典
2. 数组，可以保存一系列的值
3. 具体值，它可以是字符串，数字，json对象，数组，bool值，还可能是null

SwiftyJSON 定义了一个枚举来抽象表示这些数据类型：

``` swift
public enum Type :Int{
    case Number      
    case String
    case Bool
    case Array
    case Dictionary
    case Null
    case Unknown     // 代表未知类型，或无法解析
}
```

SwiftyJSON 使用一个私有变量`_type`来确定数据的类型，并使用一系列私有的变量来保存相应类型的元数据(Raw Value)。

``` swift
private var _type: Type = .Null  // 初始为空类型

private var rawArray: [AnyObject] = []
private var rawDictionary: [String : AnyObject] = [:]
private var rawString: String = ""
private var rawNumber: NSNumber = 0
private var rawNull: NSNull = NSNull()

// 使用私有的 _error 来存储错误
private var _error: NSError? = nil    
```

现在，可以使用一个抽象的公有变量`object`作为接口，根据*Type*枚举来设置或取得JSON相应类型的数据：

``` swift
public var object: AnyObject {
    get {
        // 根据 _type 来取得相应的数据 
    }
    set {
        // 根据 _type 来设置相应的数据 
    }
}
```

我们以`object`的`set`方法为例来演示如何设置JSON的数据：

``` swift
public var object: AnyObject {
    get {
    
    }
    set {
        _error = nil
        switch newValue {
            case let number as NSNumber:
                if number.isBool {
                    _type = .Bool
                } else {
                    _type = .Number
                }
                self.rawNumber = number
            case  let string as String:
                _type = .String
                self.rawString = string
            case  _ as NSNull:
                _type = .Null
            case let array as [AnyObject]:
                _type = .Array
                self.rawArray = array
            case let dictionary as [String : AnyObject]:
                _type = .Dictionary
                self.rawDictionary = dictionary
            default:
                _type = .Unknown
                _error = NSError(domain: ErrorDomain, code: ErrorUnsupportedType, 
                userInfo: [NSLocalizedDescriptionKey: "It is a unsupported type"])
        }
    }
}
```

`object`的`get`方法也是同样使用*switch/case*语句来根据`_type`得到相应的数据。

### 初始化

SwiftyJSON 提供了丰富的初始化方法：

``` swift
// 使用 NSData 来创建 JSON 
public init(data: NSData, options opt: NSJSONReadingOptions, error: NSErrorPointer)

// 使用拥有字典、数组等数据类型属性的对象来创建 JSON 
public init(_ object: AnyObject)

// 使用 [JSON] 来初始化 JSON
public init(_ jsonArray: [JSON])

// 使用 [String: JSON] 来初始化 JSON
public init(_ jsonDictionary: [String: JSON])

// 使用 JSON 字符串来初始化 JSON
public static func parse(string: String) -> JSON
```

第一个初始化方法使用`NSJSONSerialization`创建包含相应属性的 JSON 对象：

``` swift
public init(data:NSData, options opt: NSJSONReadingOptions = .AllowFragments, error: NSErrorPointer = nil) {
    do {
        let object: AnyObject = try NSJSONSerialization.JSONObjectWithData(data, options: opt)
        self.init(object)
    } catch let aError as NSError {
        if error != nil {
            error.memory = aError
        }
        self.init(NSNull())
    }
}
```

通过调用这个初始化方法，可以将 JSON 字符串解析为 JSON 对象。如果字符串是合法的 JSON 字符串，那么就返回相应的 JSON 对象，负责返回一个空 JSON 对象：

``` swift
public static func parse(string: String) -> JSON {
    return string.dataUsingEncoding(NSUTF8StringEncoding)
        .flatMap({JSON(data: $0)}) ?? JSON(NSNull())
}
```

剩下的三个初始化方法均是直接设置 `struct JSON` 中的 `object` 对象来完成初始化：

``` swift
public init(_ object: AnyObject) {
    self.object = object
}

public init(_ jsonArray:[JSON]) {
    self.init(jsonArray.map { $0.object })
}

public init(_ jsonDictionary:[String: JSON]) {
    var dictionary = [String: AnyObject](minimumCapacity: jsonDictionary.count)
    for (key, json) in jsonDictionary {
        dictionary[key] = json.object
    }
    self.init(dictionary)
}
```

#### 用字面量初始化 JSON 

SwiftyJSON 的另外一项功能就是可以使用字面量初始化 `struct JSON`：

``` swift
// StringLiteralConvertible
let json: JSON = "I'm a json"

// IntegerLiteralConvertible
let json: JSON =  12345

// FloatLiteralConvertible
let json: JSON =  2.8765

// With subscript in array
var json: JSON =  [1,2,3]

// With subscript in dictionary
var json: JSON =  ["name": "Jack", "age": 25]

// Array & Dictionary
var json: JSON =  ["name": "Jack", "age": 25, "list": ["a", "b", "c", ["what": "this"]]]
```

这个功能的实现很直接，就是根据[Swift Literal Convertibles](http://nshipster.com/swift-literal-convertible/)遵循一系列协议：

``` swift
extension JSON: StringLiteralConvertible { ... }

extension JSON: IntegerLiteralConvertible { ... }

extension JSON: BooleanLiteralConvertible { ... }

extension JSON: FloatLiteralConvertible { ... }

extension JSON: DictionaryLiteralConvertible { ... }

extension JSON: ArrayLiteralConvertible { ... }

extension JSON: NilLiteralConvertible { ... }
```

综上，SwiftyJSON 在初始化以后得到的是一个多个保存不同类型数据的 JSON 对象相互嵌套的模型：

![image.001](/images/2016-7-10/image.001.jpeg)

## 解析 JSON

### 下标

在 SwiftyJSON 中的`struct JSON`保存的可能是字典，数组，或者是具体的值，如：字符串，数字。当保存的是字典或数组时，我们需要使用分别使用字符串和数字作为下标来访问其中的数据。为了可以同时使用字符串和数字作为下标，SwiftyJSON 定义了一个协议 `JSONSubscriptType` 来代表 `struct JSON` 的下标，并定义枚举类型 `JSONKey` 来区分字符串和数组。

``` swift
public enum JSONKey {
    case Index(Int)
    case Key(String)
}

public protocol JSONSubscriptType {
    var jsonKey:JSONKey { get }
}

extension Int: JSONSubscriptType {
    public var jsonKey:JSONKey {
        return JSONKey.Index(self)
    }
}

extension String: JSONSubscriptType {
    public var jsonKey:JSONKey {
        return JSONKey.Key(self)
    }
}
```

现在使得字符串和整数均遵循 `JSONSubscriptType` 协议后，我们就可以同时使用这两种类型作为下标来访问数据了。SwiftyJSON 也在拓展中定义了两个下标方法，它们分别接受整数和字符串作为参数来访问数据：

``` swift
extension JSON {
    private subscript(index index: Int) -> JSON {
        get {
            if self.type != .Array {
                var r = JSON.null
                r._error = self._error ?? NSError(domain: ErrorDomain, code: ErrorWrongType, userInfo: [NSLocalizedDescriptionKey: "Array[\(index)] failure, It is not an array"])
                return r
            } else if index >= 0 && index < self.rawArray.count {
                return JSON(self.rawArray[index])
            } else {
                var r = JSON.null
                r._error = NSError(domain: ErrorDomain, code:ErrorIndexOutOfBounds , userInfo: [NSLocalizedDescriptionKey: "Array[\(index)] is out of bounds"])
                return r
            }
        }
        set {
            if self.type == .Array {
                if self.rawArray.count > index && newValue.error == nil {
                    self.rawArray[index] = newValue.object
                }
            }
        }
    }
    
    private subscript(key key: String) -> JSON {
        get {
            var r = JSON.null
            if self.type == .Dictionary {
                if let o = self.rawDictionary[key] {
                    r = JSON(o)
                } else {
                    r._error = NSError(domain: ErrorDomain, code: ErrorNotExist, userInfo: [NSLocalizedDescriptionKey: "Dictionary[\"\(key)\"] does not exist"])
                }
            } else {
                r._error = self._error ?? NSError(domain: ErrorDomain, code: ErrorWrongType, userInfo: [NSLocalizedDescriptionKey: "Dictionary[\"\(key)\"] failure, It is not an dictionary"])
            }
            return r
        }
        set {
            if self.type == .Dictionary && newValue.error == nil {
                self.rawDictionary[key] = newValue.object
            }
        }
    }
}
```

这两个私有方法分别使用整数和字符串作为参数来访问数据。在函数内部的访问逻辑中，它们分别判断了类型错误，整数下标是否越界，字符串下标是否存在等错误，完善得处理了在获得 JSON 数据过程中可能出现的错误。

SwiftyJSON 还定义了一个私有的下标方法来优雅地简化了对上述两个方法的调用：

``` swift
private subscript(sub sub: JSONSubscriptType) -> JSON {
    get {
        switch sub.jsonKey {
        case .Index(let index): return self[index: index]
        case .Key(let key): return self[key: key]
        }
    }
    set {
        switch sub.jsonKey {
        case .Index(let index): self[index: index] = newValue
        case .Key(let key): self[key: key] = newValue
        }
    }
}
```

这个私有的下标方法给提供的一个简单的内部接口，它判断下标的类型，从而在内部调用的时候可以不用关心下标的具体类型。

在以上三个私有方法的基础上，SwiftyJSON 定义了两个接口，使得我们可以用非常简单的语法来安全地访问数据：

``` swift
public subscript(path: [JSONSubscriptType]) -> JSON {
    get {
        return path.reduce(self) { $0[sub: $1] }
    }
    set {
        switch path.count {
        case 0:
            return
        case 1:
            self[sub:path[0]].object = newValue.object
        default:
            var aPath = path; aPath.removeAtIndex(0)
            var nextJSON = self[sub: path[0]]
            nextJSON[aPath] = newValue
            self[sub: path[0]] = nextJSON
        }
    }
}

public subscript(path: JSONSubscriptType...) -> JSON {
    get {
        return self[path]
    }
    set {
        self[path] = newValue
    }
}
```

这两个函数巧妙得使用了函数式编程的技巧，如`path.reduce(self) { $0[sub: $1] }`，使代码简洁易读，这也正是使用 Swift 写出的代码应该具备的特点：安全、简洁、高效。

使用这两个方法，我们可以这样得到 JSON 对象中的数据：

``` swift
let name = json[9]["list"]["person"]["name"]

let sameName = json[9,"list","person","name"]
```

这比起文章开头的例子，是不是简单、易读了太多呢？

### 安全地访问数据

SwiftyJSON 使用两种不同命名的只读计算变量来使我们可以根据自己的需要安全得访问数据。只读计算变量的命名遵循以下规则：

1. `type` —— 返回包含该类型的可选值，当数据错误时返回nil
2. `typeValue` —— 返回具体的值，当数据错误时返回具体类型的默认值，如[], 0等。

所以要得到 JSON 数组或字典，我们可以使用以下接口：

``` swift
// MARK: - Array
extension JSON {
    public var array: [JSON]? { get }     // 返回 Optional([JSON])
    public var arrayValue: [JSON] { get } // 返回 [JSON]
}

// MARK: - Dictionary
extension JSON {
    public var dictionary: [String: JSON]? { get }
    public var dictionaryValue: [String: JSON] { get }
}
```

要得到数组和字典中的元素，SwiftyJSON 也定义了相应的接口，它们均是计算属性，通过访问内部的数据结构来获取或设置数据：

``` swift
// MARK: - Array
extension JSON {
    //Optional [AnyObject]
    public var arrayObject: [AnyObject]? {
        get {
            switch self.type {
            case .Array:
                return self.rawArray
            default:
                return nil
            }
        }
        set {
            if let array = newValue {
                self.object = array
            } else {
                self.object = NSNull()
            }
        }
    }
}

// MARK: - Dictionary
extension JSON {
    //Optional [String : AnyObject]
    public var dictionaryObject: [String : AnyObject]? {
        get {
            switch self.type {
            case .Dictionary:
                return self.rawDictionary
            default:
                return nil
            }
        }
        set {
            if let v = newValue {
                self.object = v
            } else {
                self.object = NSNull()
            }
        }
    }
}
```

SwiftyJSON 中 `struct JSON` 内部使用*NSNumber*类型的`rawNumber`来保存bool和数字，所以SwiftyJSON 定义了返回*NSNumber*的接口，也可以通过这个接口来得到相应类型的数字，如：Int, Double, Float, Int8, Int16, Int32, Int64等。

特别注意一点，因为 JSON 本质上是一个文本文件，它的内容均是字符串。所以在我们要获得数字的时候，要将用字符串表示的数字，如“123”，转化为*NSNumber*。

``` swift
// MARK: - Number
extension JSON {

    //Optional number
    public var number: NSNumber? {
        get {
            switch self.type {
            case .Number, .Bool:
                return self.rawNumber
            default:
                return nil
            }
        }
        set {
            self.object = newValue ?? NSNull()
        }
    }

    //Non-optional number
    public var numberValue: NSNumber {
        get {
            switch self.type {
            case .String:
                let decimal = NSDecimalNumber(string: self.object as? String)
                if decimal == NSDecimalNumber.notANumber() {  // indicates parse error
                    return NSDecimalNumber.zero()
                }
                return decimal
            case .Number, .Bool:
                return self.object as? NSNumber ?? NSNumber(int: 0)
            default:
                return NSNumber(double: 0.0)
            }
        }
        set {
            self.object = newValue
        }
    }
}
```

之后，SwiftJSON 使用`public var number`和`public var numberValue`这两个接口在得到各种数字类型，以*Double*为例：

``` swift
extension JSON {
    public var double: Double? {
        get {
            return self.number?.double
        }
        set {
            if let newValue = newValue {
                self.object = NSNumber(double: newValue)
            } else {
                self.object = NSNull()
            }
        }
    }
    
    public var doubleValue: Double {
        get {
            return self.numberValue.doubleValue
        } 
        set {
            self.object = NSNumber(double: newValue)
        }
    }
}
```

获得其它类型数字的接口的实现方法和这段代码类似。

此外，SwiftyJSON 也定义了我们非常常用的返回URL的接口，它返回一个Optional：

``` swift
// MARK: - URL
extension JSON {
    public var URL: NSURL? { get }
}
```

### 使用 Raw Value

除了使用解析后的具体的值以外，SwiftyJSON 通过遵循*RawRepresentable*协议，来允许我们直接使用元数据(Raw Value)。我们可以选择直接使用 JSON 对象：

``` swift
extension JSON: RawRepresentable {
    public init?(rawValue: AnyObject) {
        if JSON(rawValue).type == .Unknown {
            return nil
        } else {
            self.init(rawValue)
        }
    }

    public var rawValue: AnyObject {
        return self.object
    }
}
```

可以直接将 JSON 对象转化为NSData:

``` swift
extension JSON {
    public func rawData(options opt: NSJSONWritingOptions = NSJSONWritingOptions(rawValue: 0)) throws -> NSData {
        guard NSJSONSerialization.isValidJSONObject(self.object) else {
            throw NSError(domain: ErrorDomain, code: ErrorInvalidJSON, userInfo: [NSLocalizedDescriptionKey: "JSON is invalid"])
        }

        return try NSJSONSerialization.dataWithJSONObject(self.object, options: opt)
    }
}

// 我们可以简单地将 JSON 对象转化为 NSData
if let data = json.rawData() {
    //Do something you want
}
```

也可以把 JSON 对象转化为字符串：

``` swift
extension JSON {
    public func rawString(encoding: UInt = NSUTF8StringEncoding, options opt: NSJSONWritingOptions = .PrettyPrinted) -> String? {
        switch self.type {
        case .Array, .Dictionary:
            do {
                let data = try self.rawData(options: opt)
                return NSString(data: data, encoding: encoding) as? String
            } catch _ {
                return nil
            }
        case .String:
            return self.rawString
        case .Number:
            return self.rawNumber.stringValue
        case .Bool:
            return self.rawNumber.boolValue.description
        case .Null:
            return "null"
        default:
            return nil
        }
    }
}

// 我们可以简单地将 JSON 对象转化为 String
if let string = json.rawString() {
    //Do something you want
}
```

SwiftyJSON 也让 `struct JSON` 遵循了*Printable*和*DebugPrintable*协议，使我们可以直接以字符串的形式输出JSON:

``` swift
extension JSON: Printable, DebugPrintable { ... }
```

## 遍历 JSON

在 SwiftyJSON 的[介绍](https://github.com/SwiftyJSON/SwiftyJSON/blob/master/README.md)中，描写了对 JSON 的遍历功能：

> 如果 JSON 是 Dictionary
> 
> ``` swift
> for (key: subJson): (String: JSON) in JSON {
>     // 其中 key 就是字典中的键
> }
> ```
> 
> 如果 JSON 是 Array
> 
> ``` swift
> for (index, json): (String: JSON) in JSON {
>     // 其中 key 就是字符串类型的 o..<json.count 的值
> }
> ```
> 

要使得 `struct JSON` 可以在*for...in*循环中被多次遍历，就需要让 `struct JSON` 遵从*CollectionType*协议。 

### 遵循 CollectionType 协议

首先，SwiftyJSON 定义了遵循*CollectionType*协议所需要的两个关联类型，它们分别代表了集合中的元素的生成器、访问集合元素时所要使用的下标类型，以及遍历时生成下一个元素的生成器:

``` swift
extension JSON: CollectionType {
    public typealias Generator = JSONGenerator
    
    public typealias Index = JSONIndex
    
    public func generate() -> JSON.Generator {
        return JSON.Generator(self)
    }
}
```

之后，实现了返回起始和结尾索引、根据索引返回相应元素的接口，并根据需要重写了默认的方法。

``` swift
extension JSON: CollectionType {
    public var startIndex: JSON.Index {
        switch self.type {
        case .Array:
            return JSONIndex(arrayIndex: self.rawArray.startIndex)
        case .Dictionary:
            return JSONIndex(dictionaryIndex: self.rawDictionary.startIndex)
        default:
            return JSONIndex()
        }
    }
    
    public var endIndex: JSON.Index {
        switch self.type {
        case .Array:
            return JSONIndex(arrayIndex: self.rawArray.endIndex)
        case .Dictionary:
            return JSONIndex(dictionaryIndex: self.rawDictionary.endIndex)
        default:
            return JSONIndex()
        }
    }
    
    public subscript (position: JSON.Index) -> JSON.Generator.Element {
        switch self.type {
        case .Array:
            return (String(position.arrayIndex), JSON(self.rawArray[position.arrayIndex!]))
        case .Dictionary:
            let (key, value) = self.rawDictionary[position.dictionaryIndex!]
            return (key, JSON(value))
        default:
            return ("", JSON.null)
        }
    }
    
    public var isEmpty: Bool {
        get {
            switch self.type {
            case .Array:
                return self.rawArray.isEmpty
            case .Dictionary:
                return self.rawDictionary.isEmpty
            default:
                return true
            }
        }
    }
    
    public var count: Int {
        switch self.type {
        case .Array:
            return self.rawArray.count
        case .Dictionary:
            return self.rawDictionary.count
        default:
            return 0
        }
    }
    
    public func underestimateCount() -> Int {
        switch self.type {
        case .Array:
            return self.rawArray.underestimateCount()
        case .Dictionary:
            return self.rawDictionary.underestimateCount()
        default:
            return 0
        }
    }
}
```

当 JSON 中保存的对象是数组或字典是，要分别使用整数和字符串作为索引来访问相应的值。所以在接口的实现中，根据索引的类型来访问相应的数据结构。

> 因为 Swift 是一门多范式的“面向协议”的编程语言。在 Protocol Extension 的帮助下，我们不需要实现协议的所有方法。仅仅需要覆盖少数方法即可。其它的方法可以直接使用默认实现。

#### 实现 JSONGenerator

在使 `struct JSON` 遵循*CollectionType*的时候，需要使用一个元素生成器。SwiftyJSON 定义了一个 *JSONGenerator* 类型的生成器。它的元素类型为`(String, JSON)`，即一个tuple。这个tuple的第一个值用字符串表示元素在集合中的索引；第二个值则是`struct JSON`，这也使我们可以在遍历过程中方便地访问数据。

因为在 JSON 中可以遍历数组或字典，所以*JSONGenerator*中使用了标准库中的*IndexingGenerator*, *DictionaryGenerator*两个生成器来分别生成数组或字典元素，并使用一个私有变量`type`来对生成的元素的类型加以区分。

``` swift
public struct JSONGenerator: GeneratorType {
    // 生成元素的类型为(String, JSON)
    public typealias Element = (String, JSON)

    private let type: Type
    private var dictionayGenerate: DictionaryGenerator<String, AnyObject>?
    private var arrayGenerate: IndexingGenerator<[AnyObject]>?
    private var arrayIndex: Int = 0      // 当生成器生成数组元素时，保存数组索引
    
    init(_ json: JSON) {
        self.type = json.type
        if type == .Array {
            self.arrayGenerate = json.rawArray.generate()
        }else {
            self.dictionayGenerate = json.rawDictionary.generate()
        }
    }
}
```

要实现一个生成器非常简单，仅仅需要实现一个*next*方法即可：

``` swift
public mutating func next() -> JSONGenerator.Element? {
    switch self.type {
    case .Array:
        if let o = self.arrayGenerate?.next() {
            let i = self.arrayIndex
            self.arrayIndex += 1
            return (String(i), JSON(o))
        } else {
            return nil
        }
    case .Dictionary:
        if let (k, v): (String, AnyObject) = self.dictionayGenerate?.next() {
            return (k, JSON(v))
        } else {
            return nil
        }
    default:
        return nil
    }
}
```

这个方法会根据`type`来分别调用相应的生成器来生成数组或字典元素。

#### 实现 JSONIndex

在遍历 `struct JSON` 的过程中，我们需要使用索引来访问相应的元素。因为我们所遍历的可能时数组或字典，所以在*JSONIndex*中，分别声明了两个变量`arrayIndex`和`dictionaryIndex`来作为索引，并使用一个私有变量 `type` 来区分：

``` swift
public struct JSONIndex {
    let arrayIndex: Int?
    let dictionaryIndex: DictionaryIndex<String, AnyObject>?

    let type: Type

    init(){
        self.arrayIndex = nil
        self.dictionaryIndex = nil
        self.type = .Unknown
    }

    init(arrayIndex: Int) {
        self.arrayIndex = arrayIndex
        self.dictionaryIndex = nil
        self.type = .Array
    }

    init(dictionaryIndex: DictionaryIndex<String, AnyObject>) {
        self.arrayIndex = nil
        self.dictionaryIndex = dictionaryIndex
        self.type = .Dictionary
    }
}
```

要使得*JSONIndex*可以在集合中作为索引使用，它必须遵循*ForwardIndexType*协议，并实现协议中的`successor`方法：

``` swift
extension JSONIndex: ForwardIndexType {
    public func successor() -> JSONIndex {
        switch self.type {
        case .Array:
            return JSONIndex(arrayIndex: self.arrayIndex!.successor())
        case .Dictionary:
            return JSONIndex(dictionaryIndex: self.dictionaryIndex!.successor())
        default:
            return JSONIndex()
        }
    }
}
```

在这个实现中，根据遍历数据结构类型的不同，而返回不同类型的索引。

我们需要知道两个*JSONIndex*是否相等，或得到两个索引的大小关系，所以*JSONIndex*遵循了*Equatable*和*Comparable*协议：

``` swift
extension JSONIndex: Equatable, Comparable {}

public func ==(lhs: JSONIndex, rhs: JSONIndex) -> Bool {
    switch (lhs.type, rhs.type) {
    case (.Array, .Array):
        return lhs.arrayIndex == rhs.arrayIndex
    case (.Dictionary, .Dictionary):
        return lhs.dictionaryIndex == rhs.dictionaryIndex
    default:
        return false
    }
}

public func <(lhs: JSONIndex, rhs: JSONIndex) -> Bool {
    switch (lhs.type, rhs.type) {
    case (.Array, .Array):
        return lhs.arrayIndex < rhs.arrayIndex
    case (.Dictionary, .Dictionary):
        return lhs.dictionaryIndex < rhs.dictionaryIndex
    default:
        return false
    }
}

public func <=(lhs: JSONIndex, rhs: JSONIndex) -> Bool {
    switch (lhs.type, rhs.type) {
    case (.Array, .Array):
        return lhs.arrayIndex <= rhs.arrayIndex
    case (.Dictionary, .Dictionary):
        return lhs.dictionaryIndex <= rhs.dictionaryIndex
    default:
        return false
    }
}

public func >=(lhs: JSONIndex, rhs: JSONIndex) -> Bool {
    switch (lhs.type, rhs.type) {
    case (.Array, .Array):
        return lhs.arrayIndex >= rhs.arrayIndex
    case (.Dictionary, .Dictionary):
        return lhs.dictionaryIndex >= rhs.dictionaryIndex
    default:
        return false
    }
}

public func >(lhs: JSONIndex, rhs: JSONIndex) -> Bool {
    switch (lhs.type, rhs.type) {
    case (.Array, .Array):
        return lhs.arrayIndex > rhs.arrayIndex
    case (.Dictionary, .Dictionary):
        return lhs.dictionaryIndex > rhs.dictionaryIndex
    default:
        return false
    }
}
```

## 源码学习体会

我从2014年6月，Apple在WWDC上公布Swift起就开始关注这门现代的编程语言，关注它的每一个变化，尝试用Swift来解决问题，开发App，并享受编程了乐趣。

可惜由于编程能力欠佳，我对这门语言的理解依然有限，使用Swift编写出的代码仍然浓浓的Objective-C的味道。可以说自己是在使用Swift的语法来编写Objective-C。

通过对 SwiftyJSON 这样的优秀的开源代码的学习，我学到了很多。

1. 我学会了要在在恰当地使用函数式编程，这样写出的代码简洁、高效并且有良好的可读性，SwiftyJSON 中实现*DictionaryLiteralConvertible*、实现`let name = json[9]["list"]["person"]["name"]`访问数据等都是出色的实践。
2. 对SwiftyJSON 代码的分析也加深了我对 Swift “面向协议编程”、“从协议出发编程”，值语义等概念的理解；

在今后的实践中，我也要有意识得运用这些编程技巧，写出更加Swifty的代码，更合理、高效地解决编程中遇到的问题。







