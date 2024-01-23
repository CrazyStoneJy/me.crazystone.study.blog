#### fresco图片加载流程

首先我们知道`DraweeView`继承自`ImageView`,我们可以从`DraweeView`的`onAttachedToWindow`方法开始，这是`ImageView`附着到`phoneWindow`的回调方法，之后才会调`onDraw`方法。我们可看到该方法中调用到了`DraweeView`的`doAttach`方法，而`doAttach`调用了`DraweeHolder`的`onAttach`方法，接下来我们一路追踪，发现调用到了`attachController`方法，

```java
  private void attachController() {
    if (mIsControllerAttached) {
      return;
    }
    mEventTracker.recordEvent(Event.ON_ATTACH_CONTROLLER);
    mIsControllerAttached = true;
    if (mController != null && mController.getHierarchy() != null) {
      mController.onAttach();
    }
  }
```

而其中又调用了`AbstractDraweeController`的`onAttach`方法，下面这个方法就是开始加载图片:

```java
 public void onAttach() {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("AbstractDraweeController#onAttach");
    }
    if (FLog.isLoggable(FLog.VERBOSE)) {
      FLog.v(
          TAG,
          "controller %x %s: onAttach: %s",
          System.identityHashCode(this),
          mId,
          mIsRequestSubmitted ? "request already submitted" : "request needs submit");
    }
    mEventTracker.recordEvent(Event.ON_ATTACH_CONTROLLER);
    Preconditions.checkNotNull(mSettableDraweeHierarchy);
    mDeferredReleaser.cancelDeferredRelease(this);
    mIsAttached = true;
    if (!mIsRequestSubmitted) {
      submitRequest();
    }
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
  }
```

`submitRequest`方法详情，以下是部分关键代码：

```java
protected void submitRequest() {
    ...
    // 从内存缓存中取
    final T closeableImage = getCachedImage();
    if (closeableImage != null) {
      // 如果有直接回调
      ...
      return;
    }
   	...
    // 获取getDataSource
    mDataSource = getDataSource();
    reportSubmit(mDataSource, null);
    if (FLog.isLoggable(FLog.VERBOSE)) {
      FLog.v(
          TAG,
          "controller %x %s: submitRequest: dataSource: %x",
          System.identityHashCode(this),
          mId,
          System.identityHashCode(mDataSource));
    }
    final String id = mId;
    final boolean wasImmediate = mDataSource.hasResult();
    final DataSubscriber<T> dataSubscriber =
        new BaseDataSubscriber<T>() {
          @Override
          public void onNewResultImpl(DataSource<T> dataSource) {
            // isFinished must be obtained before image, otherwise we might set intermediate result
            // as final image.
            boolean isFinished = dataSource.isFinished();
            boolean hasMultipleResults = dataSource.hasMultipleResults();
            float progress = dataSource.getProgress();
            T image = dataSource.getResult();
            if (image != null) {
              onNewResultInternal(
                  id, dataSource, image, progress, isFinished, wasImmediate, hasMultipleResults);
            } else if (isFinished) {
              onFailureInternal(id, dataSource, new NullPointerException(), /* isFinished */ true);
            }
          }

          @Override
          public void onFailureImpl(DataSource<T> dataSource) {
            onFailureInternal(id, dataSource, dataSource.getFailureCause(), /* isFinished */ true);
          }

          @Override
          public void onProgressUpdate(DataSource<T> dataSource) {
            boolean isFinished = dataSource.isFinished();
            float progress = dataSource.getProgress();
            onProgressUpdateInternal(id, dataSource, progress, isFinished);
          }
        };
    mDataSource.subscribe(dataSubscriber, mUiThreadImmediateExecutor);
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
  }
```

接下来看`getDataSource`如何获取，由于代码链路太长，debug发现，代码是调用到了`ImagePipline`的`fetchDecodedImage`方法，

```java
 public DataSource<CloseableReference<CloseableImage>> fetchDecodedImage(
      ImageRequest imageRequest,
      Object callerContext,
      ImageRequest.RequestLevel lowestPermittedRequestLevelOnSubmit,
      @Nullable RequestListener requestListener,
      @Nullable String uiComponentId) {
    try {
      // 建立prducer sequence
      Producer<CloseableReference<CloseableImage>> producerSequence =
          mProducerSequenceFactory.getDecodedImageProducerSequence(imageRequest);
      // 开始网络请求
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

