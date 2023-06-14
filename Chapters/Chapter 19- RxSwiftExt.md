# Chapter 19: RxSwiftExt

### 前言
RxSwiftExt 是RxSwfit社区主要项目下的一个项目。它有一些列`core rxswift`所没有的非常简便的操作符。

### 19.1 distinct

当处理一序列值时，如果只想遇到每个元素一次，这个操作符会非常有用。

```swift
_ = Observable.of("a", "b", "a", "c", "b", "a", "d")
  .distinct()
  .toArray()
  .subscribe(onNext: { print($0) })
  
 // 输出： ["a","b","c","d"]
```

### 19.2 mapAt

用提供的keypath，映射每个元素到值。（貌似keypath的现在可以直接用作闭包了，这个操作符应该重复了）

```swift
struct Person {
    let name: String
}

Observable
    .of(
        Person(name: "Bart"),
        Person(name: "Lisa"),
        Person(name: "Maggie")
    )
    .mapAt(\.name)
    .subscribe { print($0) }
    
    
//next(Bart)
//next(Lisa)
//next(Maggie)
//completed    
```


### 19.3 retry and repeatWithBehavior

- retry 当流遇到错误时，会使用这个retry机制，直到这个流【结束】（包含错误或者正常）。

```swift
// in case of an error initial delay will be 1 second,
// every next delay will be doubled
// delay formula is: initial * pow(1 + multiplier, Double(currentAttempt - 1)), so multiplier 1.0 means, delay will doubled
_ = sampleObservable.retry(.exponentialDelayed(maxCount: 3, initial: 1.0, multiplier: 1.0), scheduler: delayScheduler)
    .subscribe(onNext: { event in
        print("Receive event: \(event)")
    }, onError: { error in
        print("Receive error: \(error)")
    })
    
```

- repeatWithBehavior 当流【完成】时，会使用这个repeatWithBehavior机制，再次发射元素。参数跟retry基本上一样的。详情看github文档，写的非常通俗易懂。



### 19.4 [catchErrorJustComplete](https://github.com/RxSwiftCommunity/RxSwiftExt/#catcherrorjustcomplete)

Completes a sequence when an error occurs, dismissing the error condition
看文档上写的是，当错误发生的时候，不会触发error，而是直接结束流。

```swift
let _ = sampleObservable
    .do(onError: { print("Source observable emitted error \($0), ignoring it") })
    .catchErrorJustComplete()
    .subscribe {
        print ("\($0)")
}

next(First)
next(Second)
Source observable emitted error fatalError, ignoring it
completed
```


### 19.5 pausable

- `pausable` 暂停源可观察序列中的元素，除非第二个可观察序列中最新的元素为真。

```swift
let observable = Observable<Int>.interval(1, scheduler: MainScheduler.instance)

let trueAtThreeSeconds = Observable<Int>.timer(3, scheduler: MainScheduler.instance).map { _ in true }
let falseAtFiveSeconds = Observable<Int>.timer(5, scheduler: MainScheduler.instance).map { _ in false }
let pauser = Observable.of(trueAtThreeSeconds, falseAtFiveSeconds).merge()

let pausedObservable = observable.pausable(pauser)

let _ = pausedObservable
    .subscribe { print($0) }
    
//next(2)
//next(3)
```



### 19.6 bufferWithTrigger

另外一个使用trigger的操作符，跟core中的操作符`buffer(timespan:count:scheduler)`相关联。它也会缓存源流发射出来的值，并且每次trigger流发射东西，它都会发射一个缓存了值的数组。（trigger流中的元素类型不关心。）

### 19.7 withUnretained

withUnretained(_:)操作符对传入进来的object是弱引用。在源每次发出数据的时候，都会去判断这个object是否是存在的。如果这个object依旧存在，那么withUnretained参数的上的闭包就会执行，并且它返回的值是刚才新发射出的。

```swift
var anObject: SomeClass! = SomeClass()

_ = Observable
    .of(1, 2, 3, 5, 8, 13, 18, 21, 23)
    .withUnretained(anObject)
    .debug("Combined Object with Emitted Events")
    .do(onNext: { _, value in
        if value == 13 {
            // When anObject becomes nil, the next value of the source
            // sequence will try to retain it and fail.
            // As soon as it fails, the sequence will complete.
            anObject = nil
        }
    })
    .subscribe()
```
当你在闭包中想安全的捕获VC自身时，这个操作符在VC中非常有用。避免了guard的判断。

```swift
message
  .withUnretained(self) { vc, message in 
    vc.showMessage(message)
  }
```

### 19.8 partition

用于过滤。在过滤的时候，可以将流分成两个，一个是符合条件的，一个是不符合条件的。

```swift
“let (evens, odds) = Observable
                      .of(1, 2, 3, 5, 6, 7, 8)
                      .partition { $0 % 2 == 0 }

_ = evens.debug("evens").subscribe() // Emits 2, 6, 8
_ = odds.debug("odds").subscribe() // Emits 1, 3, 5, 7

```

### 19.9 mapMany
mapMany是map的一个特殊化，用于集合类型。它可以对集合类型（collection-typed）的可观察流中每个独立的元素进行映射。(`It would map every individual element inside a collection-typed observable sequence`)

```swift
_ = Observable.of(
  [1, 3, 5],
  [2, 4]
)
.mapMany { pow(2, $0) }
.debug("powers of 2")
.subscribe() // Emits [2, 8, 32] and [4, 16]
```





### ToThink 

>Remember, though, that distinct() needs some sort of storage to detect duplicates, so make sure you have an idea of the variety of unique values your sequence will emit.

这句话后半段不是很懂。



### 翻译
> (1) flagship 

某组织机构的）最重要产品，最佳服务项目，主建筑物，王牌

`RxSwiftExt https://git.io/JJXNh is another project living under the RxSwiftCommunity flagship. `


> (2) accidental strong references

意外的强引用

`particularly in a language like Swift where closures easily capture `