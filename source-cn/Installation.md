## 构建 / 安装 / 运行

Rx 不包含任何的外部依赖

下面是当前支持的选项：

### 手动方式

打开 Rx.xcworkspace，选择 `RxExample` 并且打开运行。这个方式会构建所有代码并且可以运行示例App

### [CocoaPods](https://guides.cocoapods.org/using/using-cocoapods.html)

**:warning: 重要! 为了支持 tvOS CocoaPods 需要 `0.39` 版本. :warning:**

```
# Podfile
use_frameworks!

target 'YOUR_TARGET_NAME' do
    pod 'RxSwift',    '~> 2.0'
    pod 'RxCocoa',    '~> 2.0'
    pod 'RxBlocking', '~> 2.0'
    pod 'RxTests',    '~> 2.0'
end
```

替换 `YOUR_TARGET_NAME`, 然后在 `Podfile` 目录下键入:

```
$ pod install
```

### [Carthage](https://github.com/Carthage/Carthage)

**需要Xcode 7.1**

增加配置到 `Cartfile`， 并且运行

```
github "ReactiveX/RxSwift" ~> 2.0
```

```
$ carthage update
```

### 手动利用git submodules 方式

* 将 RxSwift 增加为一个 submodule

```
$ git submodule add git@github.com:ReactiveX/RxSwift.git
```


* 拖拽 `Rx.xcodeproj` 到项目的导航栏中
* 找到 `Project > Targets > Build Phases > Link Binary With Libraries`, 点击 `+` 号 并且选择 `RxSwift-[Platform]` 和 `RxCocoa-[Platform]` 目标