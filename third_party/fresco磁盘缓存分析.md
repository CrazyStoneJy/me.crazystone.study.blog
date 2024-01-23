#### fresco磁盘缓存分析

经过网络请求结束之后，会到`DiskCacheWriteProducer.DiskCacheWriteConsumer`中进行处理。

```java
if (imageRequest.getCacheChoice() == ImageRequest.CacheChoice.SMALL) {
        mSmallImageBufferedDiskCache.put(cacheKey, newResult);
      } else {
        mDefaultBufferedDiskCache.put(cacheKey, newResult);
}
```

根据`DiskCache`缓存的策略，来决定是用`small disk cache`缓存，还是用`big disk cache`。

接下来，我们分析默认的`mDefaultBufferedDiskCache`缓存，

- step1: 将`cacheKey`&`encodeImage`按键值对的形式，放入到一个`HashMap`中。

  ```
  // Store encodedImage in staging area
  mStagingArea.put(key, encodedImage);
  ```

- Step2:开启一个线程，异步执行磁盘缓存。

  ```java
   mWriteExecutor.execute(
              new Runnable() {
                @Override
                public void run() {
                  final Object currentToken = FrescoInstrumenter.onBeginWork(token, null);
                  try {
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
  ```

接下来主要分析`writeToDiskCache`这个方法的实现逻辑.

``writeToDiskCache`调用了`FileCache#insert`方法。

```java
 mFileCache.insert(
          key,
          new WriterCallback() {
            @Override
            public void write(OutputStream os) throws IOException {
              mPooledByteStreams.copy(encodedImage.getInputStream(), os);
            }
          });
```

接下来，继续跟踪调用关系。

```java
public BinaryResource insert(CacheKey key, WriterCallback callback) {
  	// 使用SHA-1算法将cacheKey SHA加密，加密之后再使用base64算法进行encode
   resourceId = CacheKeyUtil.getFirstResourceId(key);
   DiskStorage.Inserter inserter = startInsert(resourceId, key);
      try {
        inserter.writeData(callback, key);
        // Committing the file is synchronized
        BinaryResource resource = endInsert(inserter, key, resourceId);
        cacheEvent.setItemSize(resource.size()).setCacheSize(mCacheStats.getSize());
        mCacheEventListener.onWriteSuccess(cacheEvent);
        return resource;
  }
}
```

继续看`startInsert`方法。

```java
 private DiskStorage.Inserter startInsert(final String resourceId, final CacheKey key)
      throws IOException {
    maybeEvictFilesInCacheDir();
    return mStorage.insert(resourceId, key);
  }
```



##### `DiskStorageCache#insert`

```java
public BinaryResource insert(CacheKey key, WriterCallback callback) throws IOException {
    // Write to a temp file, then move it into place. This allows more parallelism
    // when writing files.
    SettableCacheEvent cacheEvent = SettableCacheEvent.obtain().setCacheKey(key);
    mCacheEventListener.onWriteAttempt(cacheEvent);
    String resourceId;
    synchronized (mLock) {
      // for multiple resource ids associated with the same image, we only write one file
      resourceId = CacheKeyUtil.getFirstResourceId(key);
    }
    cacheEvent.setResourceId(resourceId);
    try {
      // getting the file is synchronized
      DiskStorage.Inserter inserter = startInsert(resourceId, key);
      try {
        // 通过回调将下载之后的文件流写入到.tmp文件中
        inserter.writeData(callback, key);
        // Committing the file is synchronized
        // 将.tmp文件重命名为.cnt文件
        BinaryResource resource = endInsert(inserter, key, resourceId);
        cacheEvent.setItemSize(resource.size()).setCacheSize(mCacheStats.getSize());
        mCacheEventListener.onWriteSuccess(cacheEvent);
        return resource;
      } finally {
        if (!inserter.cleanUp()) {
          FLog.e(TAG, "Failed to delete temp file");
        }
      }
    } catch (IOException ioe) {
      cacheEvent.setException(ioe);
      mCacheEventListener.onWriteException(cacheEvent);
      FLog.e(TAG, "Failed inserting a file into the cache", ioe);
      throw ioe;
    } finally {
      cacheEvent.recycle();
    }
  }

```





##### 对`maybeEvictFilesInCacheDir`函数的分析

这里调用了`maybeEvictFilesInCacheDir`方法，主要是用来在生成文件前，看`disk`的存储空间还够不够用，如果超出最大的`maxDiskSize`,则保留90%的缓存文件。

```java
private void maybeEvictFilesInCacheDir() throws IOException {
    synchronized (mLock) {
      boolean calculatedRightNow = maybeUpdateFileCacheSize();

      // Update the size limit (mCacheSizeLimit)
      updateFileCacheSizeLimit();

      long cacheSize = mCacheStats.getSize();
      // If we are going to evict force a recalculation of the size
      // (except if it was already calculated!)
      if (cacheSize > mCacheSizeLimit && !calculatedRightNow) {
        mCacheStats.reset();
        maybeUpdateFileCacheSize();
      }

      // If size has exceeded the size limit, evict some files
      if (cacheSize > mCacheSizeLimit) {
        evictAboveSize(
            mCacheSizeLimit * 9 / 10, CacheEventListener.EvictionReason.CACHE_FULL); // 90%
      }
    }
  }
```

这里超过限制主要是通过这个`evictAboveSize`方法来进行回收资源的,可看到会保留原先90%的缓存。

```javva
private void evictAboveSize(long desiredSize, CacheEventListener.EvictionReason reason)
      throws IOException {
    Collection<DiskStorage.Entry> entries;
    try {
    	// 将`entry.timestamp`大于当前时间的放在最前面；小于当前时间的，则根据`timestamp`从小到大进行排序，最后是一个这样的list
    	// [NOW + 2, Now + 1, Now +9, now - 50, now - 40, now - 30]类似这样的一个list
      entries = getSortedEntries(mStorage.getEntries());
    } catch (IOException ioe) {
      mCacheErrorLogger.logError(
          CacheErrorLogger.CacheErrorCategory.EVICTION,
          TAG,
          "evictAboveSize: " + ioe.getMessage(),
          ioe);
      throw ioe;
    }

    long cacheSizeBeforeClearance = mCacheStats.getSize();
    long deleteSize = cacheSizeBeforeClearance - desiredSize;
    int itemCount = 0;
    long sumItemSizes = 0L;
    for (DiskStorage.Entry entry : entries) {
    	// 如果删除的总size大于要删除的size，则跳出循环。
      if (sumItemSizes > (deleteSize)) {
        break;
      }
      // 删除其相应的.cnt文件
      long deletedSize = mStorage.remove(entry);
      mResourceIndex.remove(entry.getId());
      if (deletedSize > 0) {
        itemCount++;
        sumItemSizes += deletedSize;
        SettableCacheEvent cacheEvent =
            SettableCacheEvent.obtain()
                .setResourceId(entry.getId())
                .setEvictionReason(reason)
                .setItemSize(deletedSize)
                .setCacheSize(cacheSizeBeforeClearance - sumItemSizes)
                .setCacheLimit(desiredSize);
        mCacheEventListener.onEviction(cacheEvent);
        cacheEvent.recycle();
      }
    }
    mCacheStats.increment(-sumItemSizes, -itemCount);
    // 回收临时文件.tmp, 将大于当前时间30分钟的.tmp文件全部删除
    mStorage.purgeUnexpectedResources();
  }
```



##### `DefaultDiskStorage#insert`方法

```java
 public Inserter insert(String resourceId, Object debugInfo) throws IOException {
    // ensure that the parent directory exists
    FileInfo info = new FileInfo(FileType.TEMP, resourceId);
   	// 文件名/data/user/0/packageName/cache/image_cache/v2.ols100.1/xx
   	// xx为resourceId的hashCode % 100 得到的 
    File parent = getSubdirectory(info.resourceId);
    if (!parent.exists()) {
      mkdirs(parent, "insert");
    }

    try {
      // 生成.tmp文件 /data/user/0/me.crazystone.study.myapplication/cache/image_cache/v2.ols100.1/25/ySVN2K9Lo91qlMuDH_iuUylBizs.6810887363095206913.tmp
      File file = info.createTempFile(parent);
      return new InserterImpl(resourceId, file);
    } catch (IOException ioe) {
      mCacheErrorLogger.logError(
          CacheErrorLogger.CacheErrorCategory.WRITE_CREATE_TEMPFILE, TAG, "insert", ioe);
      throw ioe;
    }
  }
```



1. 生成要存放文件的文件夹

   ```java
   private String getSubdirectoryPath(String resourceId) {
       String subdirectory = String.valueOf(Math.abs(resourceId.hashCode() % SHARDING_BUCKET_COUNT));
       return mVersionDirectory + File.separator + subdirectory;
   }
   ```



/data/user/0/me.crazystone.study.myapplication/cache/image_cache/v2.ols100.1/25

```java
writeToDiskCache
```





#### 参考文档

https://juejin.cn/post/6844903559280984071