#### C++与js端的bridge绑定

js端定义global变量__fbBatchedBridge, 其value是的`MessageQueue`

```js
'use strict';

const MessageQueue = require('./MessageQueue');

const BatchedBridge: MessageQueue = new MessageQueue();

// Wire up the batched bridge on the global object so that we can call into it.
// Ideally, this would be the inverse relationship. I.e. the native environment
// provides this global directly with its script embedded. Then this module
// would export it. A possible fix would be to trim the dependencies in
// MessageQueue to its minimal features and embed that in the native runtime.

Object.defineProperty(global, '__fbBatchedBridge', {
  configurable: true,
  value: BatchedBridge,
});

module.exports = BatchedBridge;

```

在c++端调用`bindBridge`获取`MessageQueue`

```c++
void JSIExecutor::bindBridge() {
  std::call_once(bindFlag_, [this] {
    SystraceSection s("JSIExecutor::bindBridge (once)");
    // 
    Value batchedBridgeValue =
        runtime_->global().getProperty(*runtime_, "__fbBatchedBridge");
    if (batchedBridgeValue.isUndefined()) {
      throw JSINativeException(
          "Could not get BatchedBridge, make sure your bundle is packaged correctly");
    }

    Object batchedBridge = batchedBridgeValue.asObject(*runtime_);
    callFunctionReturnFlushedQueue_ = batchedBridge.getPropertyAsFunction(
        *runtime_, "callFunctionReturnFlushedQueue");
    invokeCallbackAndReturnFlushedQueue_ = batchedBridge.getPropertyAsFunction(
        *runtime_, "invokeCallbackAndReturnFlushedQueue");
    flushedQueue_ =
        batchedBridge.getPropertyAsFunction(*runtime_, "flushedQueue");
  });
}
```





#### 参考文档

https://yanbober.blog.csdn.net/article/details/53157456

https://litslink.com/blog/new-react-native-architecture

[C++如何调用java](https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/invocation.html)

[React Native渲染流程浅析](http://solart.cc/2017/08/20/react-native-render-seq/)

[ReactNative是如何让JS代码『变成』Android控件的？](https://mp.weixin.qq.com/s/nCONOPulY2H2iqiKzraXzw)

