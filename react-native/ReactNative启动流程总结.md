#### ReactNative启动流程

1. 创建`ReactInstanceManager`.

2. 调用`ReactRootView`#`startReactApplication()`

3. 开启异步线程，调用`createReactContext()`创建`ReactContxt`

   3.1 调用`processPackages`,将注册的`NativeModlue`或者`TuroboModule`全部加载到Map中，以便后续传入到c++侧.

   3.2 初始化`catalystInstance`,该类是Java侧与C++侧的桥梁,具体实现在`CatalystInstanceImpl`。调用`CatalystInstanceImpl`#`initializeBridge()`来加Java侧与C++侧联通，传入的参数有注册的`NativeModules`,`NativeModuleQueueThread`,`JSQueueThread`

4. 运行`jsbundle`,调用`catalystInstance.runJSBundle()`来运行，最后一路调用到C++侧，使用`jsc`来加载`jsbundle文件`,`jsbundle`可从`assets目录加载`,文件目录加载，网络加载等。

5. 再将`setupReactContext`线程加入到队列准备执行，该方法将`CatalystInstance`与`ReactRootView`绑定，将js端写的视图，加载到`ReactRootView`中来。

   5.1调用`ReactRootView.runApplication()`来进行。

   5.2 调用`catalystInstance.getJSModule(AppRegistry.class).runApplication(jsAppModuleName, appParams)`来运行`AppRegistry.js`的`runApplication()`，使得js视图显示出来。

   > `catalystInstance.getJSModule()`是使用动态代理实现的，具体实现逻辑在c++侧，最后是使用`catalystInstance.jniCallJSFunction(mModule, mMethod, arguments)`来执行jsModule的方法。