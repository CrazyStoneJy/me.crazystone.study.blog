### 性能优化相关资料整理

#### 工具与脚手架
- systrace -> ${ANDROID_HOME}/platform-tools/systrace/systrace.py
systrace录制
> $ python2 systrace -o trace.html
- prefotto 用于查看trace文件.

#### App Size优化


#### App进程启动优化


#### 页面启动优化


#### 页面顺畅度优化


#### APM相关开源项目
- [Tecent Matrix](https://github.com/Tencent/matrix)
- [xCrash](https://github.com/iqiyi/xCrash)
- [Kwai KOOM (oom监控)](https://github.com/KwaiAppTeam/KOOM)
- [leakCanary (内存泄漏监控)](https://github.com/square/leakcanary)

#### ANR监控
- [今日头条 ANR 优化实践系列 - 设计原理及影响因素](https://mp.weixin.qq.com/s/ApNSEWxQdM19QoCNijagtg)
- [微信Android客户端的ANR监控方案](https://mp.weixin.qq.com/s/fWoXprt2TFL1tTapt7esYg)
- [微信Android客户端的卡顿监控方案](https://mp.weixin.qq.com/s/3dubi2GVW_rVFZZztCpsKg)

#### native & so文件崩溃监控
- [Android 平台 Native 代码的崩溃捕获机制及实现](https://mp.weixin.qq.com/s/g-WzYF3wWAljok1XjPoo7w)

#### js引擎 & js代码崩溃检测
- [jsc引擎检测]
- [hermes引擎检测]
- [js代码崩溃捕获]
- [hybird chromium引擎崩溃捕获]


##### 所需相关技术
- [xHook (Android PLT hook)](https://github.com/iqiyi/xHook)


#### 相关blog
- [android performace blog](https://androidperformance.com/)
- [性能优化优秀博客站点](https://github.com/Knight-ZXW/Awesome-APM)
- [GitYuan](http://gityuan.com/)
- [青梅](https://github.com/qingmei2/blogs)

#### 其他
- 张绍文的android开发高手课