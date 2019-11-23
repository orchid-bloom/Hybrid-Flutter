####准备工作
首先，我们分别用 Xcode 与 Android Studio 快速建立一个只有首页的基本工程，工程名分别为 iOSDemo 与 AndroidDemo.

这时，Android 工程就已经准备好了；而对于 iOS 工程来说，由于基本工程并不支持以组件化的方式管理项目，因此我们还需要多做一步，将其改造成使用 CocoaPods 管理的工程，也就是要在 iOSDemo 根目录下创建一个只有基本信息的 Podfile 文件：

```
target 'iOSDemo' do
  use_frameworks!

  target 'iOSDemoTests' do
    inherit! :search_paths
  end

  target 'iOSDemoUITests' do
  end
end
```
然后，在命令行输入 pod install 后，会自动生成一个 iOSDemo.xcworkspace 文件，这时我们就完成了 iOS 工程改造。

####Flutter 混编方案介绍
如果你想要在已有的原生 App 里嵌入一些 Flutter 页面，有两个办法：

+ 将原生工程作为 Flutter 工程的子工程，由 Flutter 统一管理。这种模式，就是统一管理模式。
+ 将 Flutter 工程作为原生工程共用的子模块，维持原有的原生工程管理方式不变。这种模式，就是三端分离模式。

![flutter.png](https://upload-images.jianshu.io/upload_images/1419920-ed371f7b2542e759.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由于 Flutter 早期提供的混编方式能力及相关资料有限，国内较早使用 Flutter 混合开发的团队大多使用的是统一管理模式。

但是，随着功能迭代的深入，这种方案的弊端也随之显露，不仅三端（Android、iOS、Flutter）代码耦合严重，相关工具链耗时也随之大幅增长，导致开发效率降低。

所以，后续使用 Flutter 混合开发的团队陆续按照三端代码分离的模式来进行依赖治理，实现了 Flutter 工程的轻量级接入。

轻量级接入，三端代码分离模式把 Flutter 模块作为原生工程的子模块，还可以快速实现 Flutter 功能的解除依赖，降低原生工程的改造成本。而 Flutter 工程通过 Android Studio 进行管理，无需打开原生工程，可直接进行 Dart 代码和原生代码的开发调试。


三端工程分离模式的关键是抽离 Flutter 工程，将不同平台的构建产物依照标准组件化的形式进行管理，即 Android 使用 aar、iOS 使用 pod。也就是可以像引用其他第三方原生组件库那样快速接入 Flutter 。

####集成 Flutter

原生工程对 Flutter 的依赖主要分为两部分：
+ Flutter 库和引擎，也就是 Flutter 的 Framework 库和引擎库。
+ Flutter 工程，也就是我们自己实现的 Flutter 模块功能，主要包括 Flutter 工程 lib 目录下的 Dart 代码实现的这部分功能。

创建同级目录的Flutter module

```
Flutter create -t module flutter_demo
```

</br>
然后我们分别对 flutter_demo 进行集成
###iOS 模块集成

在 iOS 平台，原生工程对 Flutter 的依赖分别是:

+ Flutter 库和引擎，即 Flutter.framework；
+ Flutter 工程的产物，即 App.framework。


iOS 平台的对 Flutter 模块依赖，实际上就是通过打包命令生成这两个产物，并将它们封装成一个 pod 供原生工程引用。

```
Flutter build ios --debug 
```

这条命令的作用是编译 Flutter 工程生成两个产物：Flutter.framework 和 App.framework。如果需要release，把 debug 换成 release 就可以构建 release 产物.

接下来，我们让iOSDemo依赖Flutter的编译的这两个产物，这里使用cocoapods进行依赖：

我们在/flutter_demo/.ios/Flutter目录创建FlutterEngine.podspec文件

``
pod spec create FlutterEngine
``

编辑FlutterEngine.podspec 文件依赖 Flutter.framework 和 App.framework

```
Pod::Spec.new do |spec|

  spec.name         = "FlutterEngine"
  spec.version      = "0.0.1"
  spec.summary      = "A short description of FlutterEngine."
  spec.description  = <<-DESC
	A short description of FlutterEngine.
                   DESC
  spec.homepage         = 'https://github.com/xx/FlutterEngine'
  spec.license          = { :type => 'MIT', :file => 'LICENSE' }
	spec.author             = { "tema.tian" => "temagsoft@163.com" }
	spec.source       = { :git => "", :tag => "#{spec.version}" }
  spec.ios.deployment_target = '8.0'
  spec.ios.vendored_frameworks = "App.framework", "engine/Flutter.framework"

end

```

pod lib lint 一下 Flutter组件模块就做好了，我们再修改一下iOSDemo的 Podfile,把它集成进去

```
target 'iOSDemo' do
  use_frameworks!
  
  pod 'FlutterEngine', :path => '../flutter_demo/.ios/Flutter/'
  
  target 'iOSDemoTests' do
    inherit! :search_paths
  end

  target 'iOSDemoUITests' do
  end

end

```

pod install 一下，Flutter 模块就集成进 iOS 原生工程中了。

我们在修改一下iOSDemo AppDelegate 把window的rootViewController 设置为 FlutterViewController

```
  func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    window = UIWindow(frame: UIScreen.main.bounds)
    let flutterVC = FlutterViewController()
    flutterVC.setInitialRoute("defaultRoute")
    window?.rootViewController = flutterVC
    window?.makeKeyAndVisible()
    return true
  }
```

点击xcode运行，最后点击运行，官方的 Flutter Widget 也展示出来了。至此，iOS 工程的接入就完了。


![](/Users/app/Desktop/ios_demo_s.png)

###Android 模块集成
Android 原生工程对 Flutter 的依赖主要分为两部分，对应到 Android 平台，这两部分分别是：

+ Flutter 库和引擎，也就是 icudtl.dat、libFlutter.so，还有一些 class 文件。这些文件都封装在 Flutter.jar 中。
+ Flutter 工程产物，主要包括应用程序数据段 isolate_snapshot_data、应用程序指令段 isolate_snapshot_instr、虚拟机数据段 vm_snapshot_data、虚拟机指令段 vm_snapshot_instr、资源文件 Flutter_assets。

我们对 Android 的 Flutter 依赖进行抽取，首先我们再flutter_demo根目录，执行arr打包命令

```
Flutter build apk --debug
```

如果需要release，把 debug 换成 release 就可以构建 release 产物.

打包构建的flutter-debug.arr 位于 /.android/Flutter/build/outputs/aar/目录下，我们把它拷贝到AndroidDemo的App/libs目录下，然后在build.gradle中对他添加依赖：

```
 implementation(name: 'flutter-debug', ext: 'aar')

```

sync同步一下，然后我们修改AndroidDemo MainActivity的代码，将setContentView的加载View换成FlutterView

```
View FlutterView = Flutter.createView(this, getLifecycle(), "defaultRoute");
setContentView(FlutterView);
```

最后点击运行， Flutter Widget 就展示出来了.

![](/Users/app/Desktop/android_demo_s.png)
