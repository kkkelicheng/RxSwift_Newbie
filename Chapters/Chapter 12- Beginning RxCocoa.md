# Chapter 12: Beginning RxCocoa

### 前言

在前面的章节，介绍了rxswift的基础：它的作用，怎么创建，怎么订阅，怎么释放流。
理解这些很重要的知识点是为了在你的项目中更好的利用RxSwift，并且避免不想要的副作用和不想要的结果。

RxCocoa适用于所有平台，针对每个平台的需求：iOS、watchOS、iPadOS、tvOS、macOS和Mac Catalyst。每个平台都有一组自定义包装器，为许多UI控件和其他SDK类提供一组内置扩展。在本章中，您将使用iPhone和iPad上为iOS提供的那些。

总之。前面学的基础很重要，这章，将介绍另外一个framework，是Rxswift仓库的一部分： RxCocoa。


### 12.1 RxCocoa 架构

```ruby
  - RxCocoa (5.1.1):
    - RxRelay (~> 5)
    - RxSwift (~> 5)
```

[架构](https://cocoapods.org/pods/RxCocoa)

```
┌──────────────┐    ┌──────────────┐
│   RxCocoa    ├────▶   RxRelay    │
└───────┬──────┘    └──────┬───────┘
        │                  │        
┌───────▼──────────────────▼───────┐
│             RxSwift              │
└───────▲──────────────────▲───────┘
        │                  │        
┌───────┴──────┐    ┌──────┴───────┐
│    RxTest    │    │  RxBlocking  │
└──────────────┘    └──────────────┘

```

- RxSwift:

 The core of RxSwift, providing the Rx standard as (mostly) defined by ReactiveX. It has no other dependencies.
 
- RxCocoa: 

 Provides Cocoa-specific capabilities for general iOS/macOS/watchOS & tvOS app development, such as Shared Sequences, Traits, and much more. It depends on both RxSwift and RxRelay.

- RxRelay: 

 Provides PublishRelay, BehaviorRelay and ReplayRelay, three simple wrappers around Subjects. It depends on RxSwift.

- RxTest and RxBlocking: 

 Provides testing capabilities for Rx-based systems. It depends on RxSwift.


在日常的项目中，你会非常多的用到围绕UITextField和UILabel的响应式包装。

大概看看UITextField+Rx.swift中的内容，很简单的几行。

```swift
extension Reactive where Base: UITextField {
    /// Reactive wrapper for `text` property.
    public var text: ControlProperty<String?> {
        return value
    }
    
    /// Reactive wrapper for `text` property.
    public var value: ControlProperty<String?> {
        return base.rx.controlPropertyWithDefaultEvents(
            getter: { textField in
                textField.text
            },
            setter: { textField, value in
                // This check is important because setting text value always clears control state
                // including marked text selection which is imporant for proper input 
                // when IME input method is used.
                if textField.text != value {
                    textField.text = value
                }
            }
        )
    }
    // ....
}

```

什么是`ControlProperty`？

后面你会学到，现在你只需要知道它是Subject-like的类型，可以订阅它，还可以注入值。
UITextField.Rx.text 的这个名字，就很好的让你知道什么可以被订阅。这个text意味着这个属性是直接跟UITextField的text相关联。


再打开  UILabel+Rx.swift 

```swift
extension Reactive where Base: UILabel {
    
    /// Bindable sink for `text` property.
    public var text: Binder<String?> {
        return Binder(self.base) { label, text in
            label.text = text
        }
    }

    /// Bindable sink for `attributedText` property.
    public var attributedText: Binder<NSAttributedString?> {
        return Binder(self.base) { label, text in
            label.attributedText = text
        }
    }
    
}
```
跟前面一样，你会看到跟原来类相关联的属性声明，他们2个都是`Binder`类型。

Binder是一个非常有用的结构，这个结构可以接受数据，但是不可以被订阅。
另外两个关于Binder很有趣的事是：

1. 它不会接受错误

2. 它将base对象作为弱引用，然后持有。所以你不需要处理内存泄露或者弱引用问题。


### 12.2 通过基础的UIKit的组件来使用RxCocoa 

构建假的基础数据和方法

```swift
struct Weather: Decodable {
    let cityName: String
    let temperature: Int
    let humidity: Int
    let icon: String

    static let empty = Weather(
      cityName: "Unknown",
      temperature: -1000,
      humidity: 0,
      icon: iconNameToChar(icon: "e")
    )
}

class ApiController {
  static var shared = ApiController()
  func currentWeather(for city: String) -> Observable<Weather> {
  		  return Observable.just(
          Weather(
            cityName: city,
            temperature: 20,
            humidity: 90,
            icon: iconNameToChar(icon: "01d"))
        )
  }
}
```

一般来说，订阅发生在ViewDidLoad中。不恰当的订阅位置会出现不及预期的效果。
你需要在网络请求数据并且向用户显示数据之前，去创建所有的订阅者。

```c
ApiController为ViewController提供单向的数据流
┌─────────────┐            ┌───────────────┐
│             └────Data────►               │
│   APIController          │    ViewController
│             │            │               │
└─────────────┘            └───────────────┘
```
为了获取到Data,在viewDidLoad的结尾中添加如下的代码。

```swift
ApiController.shared.currentWeather(for: "RxSwift")
  .observeOn(MainScheduler.instance)
  .subscribe(onNext: { data in
    self.tempLabel.text = "\(data.temperature)° C"
    self.iconLabel.text = data.icon
    self.humidityLabel.text = "\(data.humidity)%"
    self.cityNameLabel.text = data.cityName
  })
```
运行程序后，会有2个小问题。
1个是警告，subscribe方法返回的disposable没有使用，该结果可以让你在合适的时间点取消subscription。在这个例子中，当viewController消失后，需要取消subscription，来避免潜在的内存泄漏。
为了解决上面的问题，进行如下改进。

1. 为controller增加一个DisposeBag的属性
2. 在订阅链中的末端加入dispose(by:)方法

这会将订阅者的生命周期绑定到你的DisposeBag上，并且以及你的ViewController上。
这不仅可以防止浪费资源，并且还可以避免不希望的事件以及副作用在subscription没有disposed的时候产生。

```swift
private let bag = DisposeBag()

ApiController.shared.currentWeather(for: "RxSwift")
  .observeOn(MainScheduler.instance)
  .subscribe(onNext: { data in
    //...
  })
  .disposed(by: bag)
```

现在解决了第一个问题，然后将你的注意力解决下一个。
在之前，已经探究过了text这个属性。这个属性是ControlProperty<String?>类型的，同时遵循了ObservableType和ObserverType协议。所以你可以订阅它，也可以为它添加一个新的值。
在ViewDidLoad中添加如下：

```swift
searchCityName.rx.text.orEmpty
  .filter { !$0.isEmpty }
  .flatMap { text in
    ApiController.shared
      .currentWeather(for: text)
      .catchErrorJustReturn(.empty)
  }
```
上面的代码将会返回一个新的可观察者，并显示数据。
1. orEmpty会当text发射一个nil的时候，orEmpty将会发射空的字符串。相当于`map{ $0 ?? ""}`
2. 因为currentWeather不会接受空的值，用filter进行过滤。

通过转到正确的线程并且展示数据，来继续你之前的代码块。

```swift
  .observeOn(MainScheduler.instance)
  .subscribe(onNext: { data in
    self.tempLabel.text = "\(data.temperature)° C"
    self.iconLabel.text = data.icon
    self.humidityLabel.text = "\(data.humidity)%"
    self.cityNameLabel.text = data.cityName
  })
  .disposed(by: bag)
```
现在不管你输入什么，都会获取到之前做的假数据。

> 注意，catchErrorJustReturn会在后面章节进行解释。当API发生错误时候，需要阻止observable被dispose了。在这个例子中，你需要返回一个空的reponse，当发生错误的时候，你的app才不会别停止工作。


### 12.3 从OpenWeatherAPI中获取数据

现在使用如下的方式来获取真实的数据

```swift
func currentWeather(for city: String) -> Observable<Weather> {
  buildRequest(pathComponent: "weather", params: [("q", city)])
    .map { data in
      try JSONDecoder().decode(Weather.self, from: data)
    }
}
```
数据流程如下：

```c
┌──────────────────────────────────────────┐        ┌──────┐
│              api controller              │        │      │
│                                          │        │      │
│ ┌──────────────┐       ┌──────────────┐  │        │ view │
│ │ data from api│       │map to json   │  │  data  │      │
│ │              ├───────►              │  ├────────►      │
│ └──────────────┘       └──────────────┘  │        │ controller
│                                          │        │      │
└──────────────────────────────────────────┘        └──────┘

```

尝试一个小实验，在flatmap中移除catchErrorJustReturn，当输入一个非法城市名你将收到一个404。然后你的应用将会正确的停止工作，因为你的可观察者发生了错误，然后被dispose了


### 12.4 绑定observables

苹果在macOS上有自己的绑定框架，但是过于耦合。
RxCocoa提供一个更简单的解决方案，它仅仅依赖framework中的一些类型。
在RxCocoa中非常重要的一点是，绑定是一个单向(uni-directional)的数据流。本教程中不会涉及到双向绑定（bidirectional）。

#### 12.4.1 什么是绑定可被观察者

非常简单，例如，如下两个实体的一个链接关系。

```
┌─────────────────┐          ┌─────────────────┐
│                 │ bind(to:)│                 │
│   producer      ├──────────►    consumer     │
│                 │          │                 │
└─────────────────┘          └─────────────────┘
```
1. 一个生产者（Producer），产生值
2. 一个消费者（Consumer），处理来自生产者的值

```
┌─────────────────┐          ┌─────────────────┐
│                 │ bind(to:)│                 │
│   producer      ├──────────►    consumer     │
│                 │          │                 │
│                 │◄─   X  ──┤                 │
└─────────────────┘          └─────────────────┘
```


如果你想尝试一下双向绑定，那么你需要4个实体（2个生产者，2个消费者）来模型化双向绑定这个设计。这无疑是增加了代码的复杂度。

绑定的基础方法是`bind(to:)`，用来绑定一个可被观察者对象到另外一个消费者上。这个消费者是一个`ObserverType`。例如一个Subject可以就是一个消费者。

其实`bind(to:)`是一个别名，或者说是一个语法糖。内部其实调用的还是`subscribe(observer)`方法。使用bind更加语义和直觉化。

除了`ObserverType`这类型可以用`bind`外，`Relays`也可以用，因为它自己重载了`bind`方法，虽然`Relays`没有符合`ObserverType`协议。

#### 12.4.2 使用binding可被观察者来展示数据

现在改进之前的searchCityName数据流。

```swift
let search = searchCityName.rx.text.orEmpty
  .filter { !$0.isEmpty }
  .flatMapLatest { text in
    ApiController.shared
      .currentWeather(for: text)
      .catchErrorJustReturn(.empty)
  }
  .share(replay: 1)
  .observeOn(MainScheduler.instance)
```
上面的改进有2条。

1. 使用了flatMapLatest替代flatMap，因为需要丢掉前一次的数据。
2. 订阅前使用share(replay: 1)，来让流变的更可复用。将single-use数据源，变成了multi-use的可被观察者。

这些改进将在后面专门讲MVVM的时候进行讲解。现在你只需要知道可被观察者在Rx中是一个可以被重度重用的对象。正确的模式可以将长，难读，一次使用的可被观察者，替换为多次使用并且简单理解的可被观察者。

```
┌──────────────┐     bind       ┌─────────────────┐
│              ├────────────────►temperature label│
│              │                └─────────────────┘
│              │
│              │     bind       ┌─────────────────┐
│ weather data ├────────────────►humidity label   │
│              │                └─────────────────┘
│              │
│              │     bind       ┌─────────────────┐
│              ├────────────────►icon label       │
└──────────────┘                └─────────────────┘

```
用这小的变化，它可以处理来自不同订阅者的单一参数，映射值然后展示。例如：从共享数据流中拿到温度数据然后作为字符串。

```
search.map { "\($0.temperature)° C" }
```
这将创建一个作为温度展示的字符串返回的可被观察者。
然后将它绑定到温度label上。

```swift
search.map { "\($0.temperature)° C" }
  .bind(to: tempLabel.rx.text)
  .disposed(by: bag)
```

同理可以展示其他的label

```swift
search.map(\.icon)
  .bind(to: iconLabel.rx.text)
  .disposed(by: bag)

search.map { "\($0.humidity)%" }
  .bind(to: humidityLabel.rx.text)
  .disposed(by: bag)

search.map(\.cityName)
  .bind(to: cityNameLabel.rx.text)
  .disposed(by: bag)

```


### 12.5 使用Trait来提高代码

RxCocoa提供了更加高级的特性，使得处理Cocoa和UIKit更轻松。不仅仅是bind(to:)，它还为可被观察者对象提供了专门的实现。其中有些仅仅用来被创建出来处理UI。
Trait是一组符合ObserveType协议的对象。

非常像在第一章学习的时候遇到的RxSwift的Trait，这里RxCocoa的Trait一样很实用，但是不是一定要用。

Trait在官方文档中的描述：

> Traits... help communicate and ensure observable sequence properties across interface boundaries.

使用RxCocoa的Trait的规则可能让上面更好理解。
规则
1. 它们不会产出错误
2. 它们观察和订阅都在主调度器上
3. 它们分享源，这不奇怪，因为它们都是从SharedSequence这个实体中派生出来的。Driver自动得到share(replay:1)，Signal得到是share()

这些实体保证一些东西总是被展示到用户界面，并且能被用户界面处理。

RxCocoa的Trait有如下几种
- ControlProperty 和 ControlEvent
- Dirver
- Signal

#### 12.5.1 什么是ControlProperty和Driver


- controlProperty
	用来表示组件的可读可写属性
- controlEvent
	用来监听UI组件具体的事件，例如当编辑一个textField的时候，你按下了键盘上的return键。当一个部件（`component`）使用`UIControl.Event`来跟踪它当前的状态时候，`ControlEvent`才是可用的。
- Driver
	用来限制流，限制的范围就是前面说的3点。不出错，主线程，共享。
- Signal
	同Driver，但是订阅后流不会发射最后一个元素。

因为Driver和Signal不同的replay策略，Signal对事件（event）建模很有用，而Driver则对状态（status）建模很有用。

通常来说Trait是framework中一个可选的部分。你也可以坚持用observeable和subject，但同时你需要编程的过程中使用正确的约束。如果你想保证安全和节约时间，你可以考虑用Trait。

#### 12.5.2 使用Driver和ControlProperty改进程序

第一步就是用dirver改进之前的流。

```swift
let search = searchCityName.rx.text.orEmpty
  .filter { !$0.isEmpty }
  .flatMapLatest { text in
    ApiController.shared
      .currentWeather(for: text)
      .catchErrorJustReturn(.empty)
  }
  .asDriver(onErrorJustReturn: .empty)
```
最底下的一句话（asDriver）是关键，它会将可被观察者转成driver。onErrorJustReturn参数指定一个默认值，这个值会被用在已经被转换流发生错误的时候发出。这排除了driver本身会发出error的可能。

你可能会留意到了自动补全提供额外几种方法。

- asDriver(onErrorDriveWith:)
	你可以手动处理错误，然后依照此意图。返回一个新的dirver。
- asDriver(onErrorRecover:)
	跟另外一个driver一起使用。这将会恢复刚才出错的那个driver
	
这个时候你会发现bind(to:)会出错。因为dirver没有这个方法。取而代之的是drive方法。driver的工作原理与bind(to:)非常相似，名称的差异更好地表达了使用RxCocoa特征时的意图。

现在还有一个缺陷，就是每当你输入一个字母的时候，就会去请求一次数据，有点过于频繁。
```swift
let search = searchCityName.rx.text.orEmpty
```
替换为如下的代码：

```swift
let search = searchCityName.rx
  .controlEvent(.editingDidEndOnExit)
  .map { self.searchCityName.text ?? "" }
  // rest of your .filter { }.flatMapLatest { } continues here
```
现在应用会只会在用户点击search按钮的时候才会请求数据。
但是此时，需要注意，map中的数据，重新取回了当前输入框的text值，然后继续做余下的处理。

```
┌─────────┐      ┌────────┐      ┌───────┐
│         │      │        │      │       │
│ search  │      │  get   │      │       │
│         ├──────►        ├──────► filter│
│  press  │      │ content│      │       │
│         │      │        │      │       │
└─────────┘      └────────┘      └───┬───┘
                                     │
                                     │
                                     │                   
                 ┌───────┐       ┌───▼───┐
                 │       ◄───────┤       │
                 │       │       │       │
                 │ binding       │ flatmap
                 │       │       │       │
                 │       │       │ laest │
                 └───────┘       └───────┘
```


至此，需要总结一下本章对程序的改进。
一开始用一个单一的流（single observable）去更新整个UI。经过拆分成多个块，你已经从subscribe转到bind(to:)，然后再转到drive。并且在ViewController中重用了同一个流。这种方式让代码非常容易重用，并且很方便的生效。

最后有一个特性表

![Trait feature](https://github.com/kkkelicheng/RxSwift_Newbie/blob/main/images/trait_feature)



### 12.6 RxCocoa的Disposing 

这一章的例子中没有用捕获列表，因为demo就是在rootViewController上，结束程序就什么都没了。

在RxCocoa中使用闭包的原则跟Swift语言中的指导规则是一样的。如果需要写捕获列表，但是通常不建议用unowned，而是用weak。

### 翻译


> (1) leverage

v.  充分利用
n.  影响力。杠杆作用。

lever，杠杆，控制杆。用杠杆

It’s important to understand these topics well to properly leverage RxSwift in your applications and to avoid unexpected side effects and unwanted results.


> (2) capacity

- ability 

主要指人具有做或执行的能力，这种能力可能是天生的也可能是后天习得的。

- capacity

 指的是一种接受、持有、吸收或完成某种表达或理解事物的潜力，指人或物。capacity强调的是接受能力，或者说是人的反应能力、敏感性或天资等。
 
当作能力讲的时候，capacity指向天生的能力，而ability指向后天习得的能力，而capability指向能力的极限。


> (3) pesky

adj. 讨厌的，麻烦的

> (4) controversial

adj.	有争议的; 引起争论的;

Binding is somewhat controversial: for example, Apple never released their binding system.

绑定有点争议：例如，苹果从未在iOS上发布名为Cocoa Bindings的绑定系统。


> (5) breeze

n.	微风; 和风; 轻而易举的事;

Beyond `bind(to:)`, it also offers specialized implementations of observables,

RxCocoa提供了更高级的功能，使使用Cocoa和UIKit变得轻而易举。
