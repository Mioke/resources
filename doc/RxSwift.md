# RxSwift在工程中的规范化应用

在工程中使用RxSwift和RxCocoa之后，遇到了不少问题，从这些问题中总结出了一些优化点和需要注意的地方，作为讨论和以后编码时的指导。

### 简化Observable功能

Observable用来描述一个操作的流程，简单可以描述为：`输入`->`执行`->`输出`*n->`结束`。对于一个Observable来说，他的`执行`部分代码需要执行尽量少的功能代码，如果功能较为庞大则可以考虑拆分为不同的功能。这样优点是显而易见的，对于函数式编程来说，功能越简单的函子(functor)越能组合出越多功能越复杂的函数。第二，这种小功能的函数也更方便测试和定位问题，同时使代码结构更好，代码可读性更强。

例子：获取邮件的操作流程如下，如果展开所有功能，代码是这样的，首先会检查网络，再检查账号登录情况，再去拉取message，返回拉取的数据。
```swift
func fetchMailsOb(of folderPath: String) -> Observable<IMAPResult<[MCOIMAPMessage]>> {
        if Utils.netIsReachable {
        // check acount:
            if IMAPManager.shared.checkAccount(username: user, password: pass) {
                // do something
                return IMAPManager.shared
                    .fetchLatestMessages(expectedNumber: kMaxMessage, folder: folder)
                    .map({ (messages) -> IMAPResult<[MCOIMAPMessage]> in
                        return IMAPResult.succ(messages)
                    })
            } else {
                return .error(Errors.CheckAccountError)
            }
        } else {
            return .just(IMAPResult.failed(Errors.Request.noNetworkError))
        }
    }
```

根据我们的简化原则，可以把网络检查、账号检查再封装成单独的Observable再进行使用：

```swift
func reachabilityChecking() -> Observable<Void> {...} // throw error when not reachable
func IMAPAccountChecking() -> Observable<Void> {...} // throw error when failed.

func fetchMailsOb(of folderPath: String) -> Observable<IMAPResult<[MCOIMAPMessage]>> {
    return reachabilityChecking()
        .flatMap { IMAPAccountChecking() }
        .flatMap { IMAPManager.shared.fetchLatestMessages(expectedNumber: kMaxMessage, folder: folder) }
        .map({ (messages) -> IMAPResult<[MCOIMAPMessage]> in
            return IMAPResult.succ(messages)
        })
        .catchError { .just(IMAPResult.failed($0)) }
}
```

这样就简化了`fetchMailOb(of:)`这个函数，可读性更强，并且网络检查、账号检测的方法同时可以提供给其他函数进行使用，提高复用性减少代码量。


### 错误处理尽量使用Observable.error

Rx给我们提供了统一的错误处理报告机制，他是一种阻断式的处理方式，在链式流里当其中一个操作发生错误，则整个链式流终止。

```swift
a.flatMap { _ in b }.flatMap { _ in c } // 如果b throw了一个error1，则Observable将收到一个Error Event(error1)。
```

统一使用的好处是，可以规范应用内的错误处理机制。如果不同模块间定义了多种错误类型，如：`MyResult.failed(Error)`, `YourResult.error(Error)`，在运算或结合函数时，难免做类型转换和判断，带来不必要的代码量。

在获取到Error之后也更容易使用`catchError`或`do(onError:)`来做一些统一处理。

- 特殊情况：
在实际需求中可能出现，a的错误则弹Toast提示，b的错误则弹Alert，c的提示则不响应。这种情况则在创建链式操作流时做区分操作：
```swift
a.do(onError: {
        toast(...)
    })
    .flatMap{ _ in b }
    .do(onError: {
        alert()
    })
    .flatMap { _ in c }
```

### 使用值类型传递数据

在实际使用函数时，经常会用到闭包，从而会涉及到引用类型和值类型的使用。引用类型则要注意循环引用引起的内存泄露。

```swift
var ref = ClassA()

operation
    .flatMap { [weak ref] rst in
        return information(with: rst.first, in: ref)
    }
```

这样的话还是会在特定条件下遇到问题，比如多线程时，多个线程同时使用引用对象`ref`，因为`Swift`中`let`的引用对象并不是线程安全的，所以可能会造成读写数据的不可靠，而且多线程下更消耗资源。如果不可避免的使用引用类型，则需要注意这些问题。

值类型则不用顾虑这么多，闭包的上下文捕捉、赋值都是基于拷贝的，并且会有编译优化。

### 使用`shareReplay()`,`share()`提高效率

如果使用非连接式的Observable，则多次订阅会造成Observable的多次调用。这时候我们不想重复触发，可以选择使用`publish()`,`replay()`,`share()`,`shareReplay()`等方式共享订阅者。

- `shareReplay(n)`
可以让观察者共享一个源，并且回放n个订阅事件。

- `share()`
与`shareReplay()`区别是，在订阅后，不会受到订阅之前的信号。

### RxRealm使用

实际使用中经常用到监听数据库对象的数据变化，从而触发某些业务逻辑或反应到界面上。

- 对于监听列表，可以使用`Observable.collection(from:synchronousStart:)`：

```swift
let realm = try! Realm()
let laps = realm.objects(Lap.self)

Observable.collection(from: laps)
  .map { 
    laps in "\(laps.count) laps"
  }
  .subscribe(onNext: { text in
    print(text)
  })
```

- 监听单个对象，或指定其某个属性：
```swift
Observable.from(object: ticker, properties: ["name", "id", "family"])
```

- 注意事项：在注册notification时（监听变化），realm会进入write transaction，所以在开始监听的时候要注意`写冲突`。

- `add()` and `delete()`

Rx式的写操作：

```swift
let realm = try! Realm()
let messages = [Message("hello"), Message("world")]

Observable.from(messages)
  .subscribe(realm.rx.add())

let allMessages = realm.objects(Message.self)
Observable.from(allMessages)
  .subscribe(realm.rx.delete())
```

- TableView, CollectionView数据绑定
 
使用库[RxRealmDataSources](https://github.com/RxSwiftCommunity/RxRealmDataSources)，可以简单的建立DataSource，直接绑定数据到TableView或CollectionView上。

```swift
// 无动画可以直接使用bindTo:
Observable.from( [Realm collection] )
  .bindTo(tableView.rx.items) {tv, ip, element in
    let cell = tv.dequeueReusableCell(withIdentifier: "Cell")!
    cell.textLabel?.text = element.text
    return cell
  }
  .addDisposableTo(bag)


// 包含动画：
// create data source
let dataSource = RxTableViewRealmDataSource<Lap>(
  cellIdentifier: "Cell", cellType: PersonCell.self) {cell, ip, lap in
    cell.customLabel.text = "\(ip.row). \(lap.text)"
}

// RxRealm to get Observable<Results>
let realm = try! Realm()
let lapsList = realm.objects(Timer.self).first!.laps
let laps = Observable.changeset(from: lapsList)

// bind to table view
laps
  .bindTo(tableView.rx.realmChanges(dataSource))
  .addDisposableTo(bag)
```

### 线程使用

如果某些操作过于耗时或消耗资源，我们通常会建立独立的线程用来执行这些工作。在Rx中我们使用`observeOn:`和`subscribeOn:`来分配工作在指定的线程：

`observeOn:`和`subscribeOn:`的区别是，`observeOn`只保证观察对象的回调在指定的线程，而`subscribeOn:`则会把订阅和非订阅的逻辑都放在指定线程执行。

```swift
let a = Observable.just(1)
let b = Observable.just(2)

a
    .flatMap { _ -> Observable<Int> in
        print("logic on: \(Thread.current)")
        return b
    }
    .subscribeOn(ConcurrentDispatchQueueScheduler.init(qos: .default))
    .subscribe { (event) in
        print("\(event), \(Thread.current)")
    }
    .disposed(by: bag)

// output:
logic on: <NSThread: 0x600000264040>{number = 3, name = (null)}
next(2), <NSThread: 0x600000264040>{number = 3, name = (null)}
completed, <NSThread: 0x600000264040>{number = 3, name = (null)}


a
    .flatMap { _ -> Observable<Int> in
        print("logic on: \(Thread.current)")
        return b
    }
    .observeOn(ConcurrentDispatchQueueScheduler.init(qos: .default))
    .subscribe { (event) in
        print("\(event), \(Thread.current)")
    }
    .disposed(by: bag)
// output:
logic on: <NSThread: 0x600000074080>{number = 1, name = main}  -> 逻辑处理在主线程
next(2), <NSThread: 0x6000002611c0>{number = 5, name = (null)}
completed, <NSThread: 0x6000002611c0>{number = 5, name = (null)}
```

使用`subscribeOn:`情况下，在操作链中任意一个地方调用，预期结果都是一样的。`observeOn:`则只会影响它之后的数据操作。混合调用`subscribeOn:`和`observeOn:`会导致切换线程:

```swift
let a = Observable<Int>.create { (observer) -> Disposable in
    print("a create on: \(Thread.current)")
    observer.onNext(1)
    observer.onCompleted()
    return Disposables.create()
}

let b = Observable.just(2)

a
    .flatMap { _ -> Observable<Int> in
        print("logic on: \(Thread.current)")
        return b
    }
    .subscribeOn(ConcurrentDispatchQueueScheduler.init(qos: .default))
    .observeOn(ConcurrentDispatchQueueScheduler.init(qos: .default))
    .subscribe { (event) in
        print("\(event), \(Thread.current)")
    }
    .disposed(by: bag)

// output:
a create on: <NSThread: 0x600000266840>{number = 5, name = (null)}
logic on: <NSThread: 0x600000266840>{number = 5, name = (null)}
next(2), <NSThread: 0x600000265c80>{number = 4, name = (null)}
completed, <NSThread: 0x600000265c80>{number = 4, name = (null)}
```

- 如果要在其他线程使用realm数据库对象，则需要在闭包中重新获取对象。 

### Side-effects（副作用）

首先说明下什么是副作用。如果一个函数被认为包含副作用，则它除了返回一个值之外，还会造成其他可观察的效应。通常来说这个效应是指`状态`的改变，可能为下面情况：

- 更改了一个在大于函数当前范围（scope）的变量。
- 文件或网络的I/O操作
- 更新了界面信息

举个简单的例子：
```swift
var i = 0

let observable = Observable.from([1,2,3])
    .map { ele in
        i += 1
        return ele + i
    }

observable.subscribe(on: { event in
    if case let Event.next(result) = event {
        print("observer a:\(result)")
    }
}).disposed(by: bag)

// output:
observer a:2
observer a:4
observer a:6

observable.subscribe(on: { event in
    if case let Event.next(result) = event {
        print("observer b:\(result)")
    }
}).disposed(by: bag)

// output:
observer b:5
observer b:7
observer b:9
```

副作用有可能是偶然的，也有可能是有意为之的。简单来说避免副作用的方式是，尽量在函数内避免改变其他变量或状态。为什么要避免这种副作用？

1. 使代码更具可维护性(maintainability),一个高性能、高可维护性的函数应该确定输入和输出的逻辑性，如果函数中包含不确定的可变元素，可能导致同样的输入产出不同的结果。这对其他使用者来说会造成疑惑甚至是bug。
2. 函数改变了公共属性或状态，这些属性和状态则对于其他人来说可能是不可预知的。
3. 不方便测试。

怎么解决副作用，我们可以把状态或者值当做一个函数的一个参数来改造函数：

```swift
func increase(of array: [Int]) -> Observable<Int> {
    let indeces = 0..<array.count
    return Observable.from(indeces)
        .map { index in
            return array[index] + index
        }
}
```

这样这个函数就保证了不管谁调用，输入输出都是可控的。

### 创建易于测试的代码

下面的代码可能是我们平常经常编写的：

```swift
func someFunction() -> Observable<String> {
    return otherOb.map { result in
        return result + GlobalInstance.property1
    }
    ...
}
```

`GlobalInstance.property1`可能是工程中的全局变量，在我们意识到这个变量是测试条件之前，直接在函数中使用是合理的。

这样的代码在平常运行和阅读时没有任何问题，但是如果在测试时，`GlobalInstance.property1`成为测试条件之一时，这个方法就变得不可测试了。简单的做法是，在写一个函数时，可以优先考虑下这个函数的功能和测试条件，尽量把测试条件当做可变参数进行构建：

```swift
func someFunction(with param: String) -> Observable<String> {
    return otherOb.map { result in
        return result + param
    }
    ...
}
```

这样的话，可以根据不同的测试条件设定函数的输入来测试函数的正确性。

如果输入条件比较复杂，可以根据实际分类来规整出协议，提高可读性和可维护性：

```swift
func someFunction(with info: ProtocolA & ProtocolB) -> Observable<String> {
    return otherOb
        .map { result in
            return result + info.property1 + info.property2
    }
}

func test() {
    let scenarioA: (ProtocolA & ProtocolB) = ...
    let scenarioB: (ProtocolA & ProtocolB) = ...

    someFunction(with: scenarioA)
        .subscribe(onNext: { ele
            ...
        })
        .dispose(by: bag)

    someFunction(with: scenarioB)
        .subscribe(onNext: { ele
            ...
        })
        .dispose(by: bag)
}
```



