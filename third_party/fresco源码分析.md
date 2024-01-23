#### fresco介绍

说到android图片加载框架，我们都熟悉`glide`,`Picasso`,`fresco`等，由于我们公司使用了`react-native`作为了主要的跨平台开发框架，其中自然而然的集成了`facebook`的`fresco`。所以，在这里主要分析一下`fresco`是怎么加载图片的.

[fresco github仓库](https://github.com/facebook/fresco)

[fresco官网](https://frescolib.org/docs/)

在使用方面`fresco`是其中最复杂的，但是它的优势也是比较明显的，主要是在以下这几个方面：

- 图片的渐进式呈现。
- **支持动图加载：**加载Gif、WebP动图，每一帧都是一张很大的Bitmap，每个动画都有很多帧。Fresco能管理好每一帧并管理好你的内存。
- **丰富的图片处理：**缩放、圆角、透明、高斯模糊等处理。

- 在5.0以下系统，Bitmap缓存位于`ashmem`,这样Bitmap对象的创建和释放将不会引发GC,更少的GC会使你的App运行得更加流畅。
- 良好的代码设计，代码可扩展性非常好。



#### fresco加载图片流程

我们知道图片加载框架，为了加快下次图片的加载速度，一般是有其缓存机制的，而fresco的缓存机制也和大多数图片加载库一样，有着`memory cahce`&`disk cache`，但是`fresco`缓存策略比较复杂，分为：bitmap缓存，未解码图片的内存缓存，磁盘缓存。`fresco`有着三级缓存。

> 注解：所谓图片是否解码，指的是图片是以bitmap的形式存在，还是二进制流的形式存在。未解码表示是二进制流的形式，解码表示将二进制流转为bitmap对象。

简易的加载流程如下，

![](https://i.loli.net/2021/02/28/X4ylrFh5oaRZkwD.jpg)

因为在具体的图片加载逻辑中，设计到各种缓存,decode,裁剪图片等操作，流程是非常复杂的，下面是正常图片加载Producer和Consumer的流程。

<img src="https://i.loli.net/2021/02/28/cqzuLKveIP2Jyhd.png"/>

关于这块的逻辑，fresco设计的非常巧妙，使用装饰器和责任链模式混用的方式，使得代码结构变得非常容易扩展，而且想要自己定义图片的加载顺序，只需要按照相应的顺序，嵌套new一个producer sequence就可以了。我们目前请求图片就是用的`getDecodedImageProducerSequence`这个sequence.

```java
 public DataSource<CloseableReference<CloseableImage>> fetchDecodedImage(
      ImageRequest imageRequest,
      Object callerContext,
      ImageRequest.RequestLevel lowestPermittedRequestLevelOnSubmit,
      @Nullable RequestListener requestListener,
      @Nullable String uiComponentId) {
    try {
      Producer<CloseableReference<CloseableImage>> producerSequence =
          mProducerSequenceFactory.getDecodedImageProducerSequence(imageRequest);
      return submitFetchRequest(
          producerSequence,
          imageRequest,
          lowestPermittedRequestLevelOnSubmit,
          callerContext,
          requestListener,
          uiComponentId);
    } catch (Exception exception) {
      return DataSources.immediateFailedDataSource(exception);
    }
  }

```

`fresco`有个`producerSequenceFactory`有许多不同的`producer sequence`，方便我们根据不同的场景选择不同的sequence。

![](https://i.loli.net/2021/02/28/G2AFPafwHIOYz31.png)

关于producer和consumer的设计，我使用ts写了一个简单的实现，可以一块预览一下。



#### bitmapMemoryCache && encode Image Cache

`bitmapMemoryCache`和`EncodedMemoryCache`是使用的相同的数据结构，`MemoryCache`来存储的。主要区别是`bitmapMemoryCache`缓存的是`CloseableImage`,而`EncodedMemoryCache`缓存的是`PooledByteBuffer`字节流。

![](https://i.loli.net/2021/03/07/Tap741mIKZO3sFe.png)

> LRUMap: Least Recently Used Map,即最近最少使用，是一种常用的页面置换算法，选择最近最久未使用的页面予以淘汰。 也常见于软件开发中，用于处理一些缓存策略。`fresco`中使用`LinkedHashMap`实现`LRU`算法，`LinkedHashMap`是`jdk`中，一个具有双链表的`Map`，是一个有序的`HashMap`,这里的有序指的是元素的插入顺序。



#### disk cache

`fresco`磁盘缓存是分为`SMALL`,`DEFAULT`两种，这两种策略的都是用`BufferedDiskCache`做的缓存，因此，直接分析这个类就可以了。

```java
 public void put(final CacheKey key, EncodedImage encodedImage) {
    try {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.beginSection("BufferedDiskCache#put");
      }
      Preconditions.checkNotNull(key);
      Preconditions.checkArgument(EncodedImage.isValid(encodedImage));

      // Store encodedImage in staging area
      mStagingArea.put(key, encodedImage);

      // Write to disk cache. This will be executed on background thread, so increment the ref
      // count. When this write completes (with success/failure), then we will bump down the
      // ref count again.
      final EncodedImage finalEncodedImage = EncodedImage.cloneOrNull(encodedImage);
      try {
        final Object token = FrescoInstrumenter.onBeforeSubmitWork("BufferedDiskCache_putAsync");
        // 开启异步线程
        mWriteExecutor.execute(
            new Runnable() {
              @Override
              public void run() {
                final Object currentToken = FrescoInstrumenter.onBeginWork(token, null);
                try {
                  // 将缓存写入到disk中
                  writeToDiskCache(key, finalEncodedImage);
                } catch (Throwable th) {
                  FrescoInstrumenter.markFailure(token, th);
                  throw th;
                } finally {
                  mStagingArea.remove(key, finalEncodedImage);
                  EncodedImage.closeSafely(finalEncodedImage);
                  FrescoInstrumenter.onEndWork(currentToken);
                }
              }
            });
      } catch (Exception exception) {
        // We failed to enqueue cache write. Log failure and decrement ref count
        // TODO: 3697790
        FLog.w(TAG, exception, "Failed to schedule disk-cache write for %s", key.getUriString());
        mStagingArea.remove(key, encodedImage);
        EncodedImage.closeSafely(finalEncodedImage);
      }
    } finally {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.endSection();
      }
    }
  }
```

首先是按照[cacheKey,EncodeImage]的方式，将其缓存在Map中，接下来会开启一个线程，用来异步存储encodeImage到disk中。在这里通过图片的url，经过SHA1加密和base64转码，之后得到一个resourceId,主要用来生成存储文件的文件名，文件名格式为：/data/user/0/package/cache/image_cache/v2.ols100.1/95/mtQL41H7RDPU2uZo1zMCo5fto-Q.4178151137210219191.tmp

![](https://i.loli.net/2021/02/28/1M9dgH4QwBcZu2X.png)

#### 图片加载网络请求

`fresco`关于图片的网络请求，抽象出来一个`NetworkFetcher`接口来处理，因此，开发者可以通过继承`NetworkFetcher`，使用自己熟悉的网络请求库来封装。`fresco`内部已经有了`volley`,`okhttp`,`httpUrlConnection`三种网络库的实现。

![](https://i.loli.net/2021/03/07/eCGHSBwL2ZnDFKQ.png)



#### ReactImageView是如何加载的？

##### 1.1`qrn`是如何加载的？

首先我们先分析一下`qrn`是如何加载的？我们打开一个rn页面，一般是通过scheme的形式打开，scheme的类似：`qunariphone://react/open?hybridId=f_home_rn&pageName=Home&initProps=${encodeURIComponent ( JSON.stringify ({"param":{"cityName":"xx”,"bd_source":"xx”}}))}`,经过native代码桥接的`QRCTJumpHandleManager`module配合动态路由，跳转到`QReactNativeActivity`页面，再调用`QReactHelper#doCreate`开始，创建rn环境。



`qrn`对于`ReactInstanceManager`的缓存处理

![](https://i.loli.net/2021/03/09/LfYPBl9pWx7Rbjk.png)

创建rn环境：

![](https://i.loli.net/2021/03/09/2PlRBDFWS9oTyed.jpg)

##### 1.2 Image标签的渲染

以上`react-native`环境是初始化完成，接下来分析一下，用react编写的`Image`是如何渲染到页面上的。

一个简单的列子，

```js
import React, { Component } from 'react';
import {
  AppRegistry,
  View,
  Image
} from 'react-native';

export default class App extends Component {
  render() {
    return (
      <View>
       <Image style={{ height: 50, width: 50, flexDirection: 'row' }} source={{ uri: '' }}/>
      </View>
    );
  }
}


AppRegistry.registerComponent('app', () => App);
```

`AppResgistry`注册完组件之后，Native端调用`AppRegistry`的`runApplication`开始渲染组件。

写过原生android的同学都知道，android一般用布局方式是`LinearLayout`,`RealativeLayout`,`FrameLayout`等，并没有所谓的`flex`布局，`facebook`使用c++实现了一套`flex`布局，即`Yoga`.

在js端，`facebook`实现了一套虚拟的dom树结构，可以在`ReactNativeRender-prod.js`查看其实现方式，在`ReactNativeRender`中，`react-native`将所有的view操作都抽象为了`UI操作`，对应的是`native side`  的`UIOpeation`,比如：创建view,更新view,测量view。对应的是`CreateViewOperation`,`UpdateViewExtraData`，`MeasureOperation`.

![](https://i.loli.net/2021/03/11/mwTxpsR4K7S6A9g.png)



大致流程如下图所示：

![](https://i.loli.net/2021/03/11/TnmLhVtSKpdo6RI.jpg)

`React Component`在native端View的映射：

![](https://i.loli.net/2021/03/11/PNFKa2LS31epzBU.jpg)

native和js端双边通信简易图：

![](https://i.loli.net/2021/03/11/uMl6X5WzgeSvfdo.jpg)



#### 参考文档

https://juejin.cn/post/6844903559280984071

[ReactNative Native层的渲染流程](https://juejin.cn/post/6844904184542822408)

[fresco 源码](https://github.com/facebook/fresco)

https://yanbober.blog.csdn.net/article/details/53157456

https://litslink.com/blog/new-react-native-architecture

[C++如何调用java](https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/invocation.html)

[React Native渲染流程浅析](http://solart.cc/2017/08/20/react-native-render-seq/)

[ReactNative源码篇：渲染原理](https://github.com/sucese/react-native/blob/master/doc/ReactNative%E6%BA%90%E7%A0%81%E7%AF%87/4ReactNative%E6%BA%90%E7%A0%81%E7%AF%87%EF%BC%9A%E6%B8%B2%E6%9F%93%E5%8E%9F%E7%90%86.md)

